# Ryu-Simple Switch with Openflow 1.3

 此程式的兩大主要功能:
 
 * 分派管轄網路內的封包流向
 * 處理不明封包

## 開頭宣告

```python
from ryu.base import app_manager
from ryu.controller import ofp_event
from ryu.controller.handler import CONFIG_DISPATCHER, MAIN_DISPATCHER
from ryu.controller.handler import set_ev_cls
from ryu.ofproto import ofproto_v1_3
from ryu.lib.packet import packet
from ryu.lib.packet import ethernet
```

#### app\_manager
實作 Ryu 應用程式，都需要繼承 app\_manager.RyuApp。

#### ryu.controller 的事件類別名稱
* CONFIG\_DISPATCHER：接收 SwitchFeatures
* MAIN\_DISPATCHER：一般狀態
* DEAD\_DISPATCHER：連線中斷 //在此沒有用到
* HANDSHAKE\_DISPATCHER：交換 HELLO 訊息 //在此沒有用到

#### set\_ev\_cls
當裝飾器使用。因 Ryu 接受到任何一個 OpenFlow 的訊息，都會需要產生一個對應的事件。為了達到這樣的目的，透過 set\_ev\_cls 當裝飾器，依接收到的參數（事件類別、 Switch 狀態），而進行反應。

#### ofproto\_v1\_3
使用的 OpenFlow 版本1.3。

## 初始化
```python
class SimpleSwitch13(app_manager.RyuApp): 
    OFP_VERSIONS = [ofproto_v1_3.OFP_VERSION]
 
    def __init__(self, *args, **kwargs):
        super(SimpleSwitch13, self).__init__(*args, **kwargs)
        self.mac_to_port = {}
    # ...
```

#### class SimpleSwitch13(app\_manager.RyuApp):
第一步就是繼承 app\_manager.RyuApp，讓 App 在 Ryu 的框架中。


#### OFP\_VERSIONS = [ofproto\_v1\_3.OFP\_VERSION]
設定 OpenFlow 的版本為1.3版。

#### super(SimpleSwit…
實體化父類別（app\_manager.RyuApp），讓此 app 的 RyuApp 功能實體化（可以使用）。

#### self.mac\_to\_port = {}
建立參數 mac\_to\_port，當作 MAC 位址表。

## 加入 Table-miss Flow Entry 事件處理

在 OpenFlow Switch 中，至少有一個```Flow Table```，每個```Flow Table```又會包含多個```Flow Entry```。這些```Flow Table```是有分優先權的，優先權的順序由0開始往上。```Flow Table```中存放的```Flow Entry```，裡面所包含的就是相對應的執行動作。當```Packet-in```的時候，就會去```Flow Table```中，找尋可以```match```的```Flow Entry```並執行其```Flow Entry```設定的執行的動作。

在這裡增加的```Table-miss Flow Entry```，就是要處裡「遇到沒辦法```match```的```Flow Entry```」該做的事。

```python
@set_ev_cls(ofp_event.EventOFPSwitchFeatures, CONFIG_DISPATCHER)
def switch_features_handler(self, ev):
    datapath = ev.msg.datapath
    ofproto = datapath.ofproto
    parser = datapath.ofproto_parser
    #...
```

#### ev.msg
儲存 OpenFlow 的對應訊息。

#### datapath
處理 OpenFlow 訊息的類別。

#### ofproto
OpenFlow 的常數模組，做為通訊協定中的常數設定使用。

#### parser
OpenFlow 的解析模組，解析模組提供各個 OpenFlow 訊息的對應類別。

> ofprotor 及 parser 的資料[參考來源](https://osrg.github.io/ryu-book/zh_tw/html/ofproto_lib.html)。

```python
def switch_features_handler(self, ev):
 
    #...
 
    match = parser.OFPMatch()
    actions = [parser.OFPActionOutput(ofproto.OFPP_CONTROLLER,ofproto.OFPCML_NO_BUFFER)]
    self.add_flow(datapath, 0, match, actions) 
```

#### match = parser.OFPMatch()
產生一個新的 match。

#### actions = [parser.OFPActio…
產生一個 OFPActionOutput 類別 action，並指定他是要傳送至 Controller 的（ofproto.OFPP_ CONTROLLER），且並不使用 Buffer（ofproto.OFPCML_ NO_BUFFER）。

#### self.add\_flow(dat…
夾帶 datapath、優先權（0）、match、actions，到我們自行寫的 add\_flow func 中（等一下文中會進行介紹），並觸發 Packet-In 事件（封包並沒有 match 到任何 Flow Entry 的情況下，就會觸發）。

> Table-miss Flow Entry 的優先權設定為0，也就是最低的優先權。也因為優先權最低，其實就代表著，同 Table 中的 Flow Entry 都沒有對應到的情況。

## 加入 Packet-In 事件處理
```python
@set_ev_cls(ofp_event.EventOFPPacketIn, MAIN_DISPATCHER)
def _packet_in_handler(self, ev):
    msg = ev.msg
    datapath = msg.datapath
    ofproto = datapath.ofproto
    parser = datapath.ofproto_parser
 
    #...
```
在Packet-In事件處理中，一開始的參數設定跟剛剛所提到的是一樣的，所以不再贅述囉！接下來直接進入```更新MAC位址表```的部分。

```python
def _packet_in_handler(self, ev):
    #...
 
    in_port = msg.match['in_port']
 
    pkt = packet.Packet(msg.data)
    eth = pkt.get_protocols(ethernet.ethernet)[0]
 
    dst = eth.dst
    src = eth.src
 
    dpid = datapath.id
    self.mac_to_port.setdefault(dpid, {})
 
    self.logger.info("packet in %s %s %s %s", dpid, src, dst, in_port)
 
    self.mac_to_port[dpid][src] = in_port
    #...   
```

#### 解析訊息，並存放至參數中

in_port、pkt、eth、dst、src、dpid。以上都是等一下，運作中會用到的參數。

#### 設定更新 MAC 表
self.mac_ to_ port.setdefault(dpid, {}) 及 self.mac_ to_ port[dpid][src] = in_port 的作用，都是用來將來源主機放入 MAC 表中。

> 會使用 dpid 當 mac_ to_port 的第一識別參數，是因為當有多個 OpenFlow 交換器時，可以使用它來識別。

```python
def _packet_in_handler(self, ev):
 
    #...
 
    if dst in self.mac_to_port[dpid]:
        out_port = self.mac_to_port[dpid][dst]
    else:
        out_port = ofproto.OFPP_FLODD
 
    actions = [parser.OFPActionOutput(out_port)]
 
    if out_port != ofproto.OFPP_FLOOD:
        match = parser.OFPMatch(in_port=in_port, eth_dst=dst)
        self.add_flow(datapath, 1, match, actions)
    #...
```
#### if dst in self.ma…
判斷目的位址是否已經在 MAC 表中，如果有，則直接將對應的位址放入```out_port```。如果沒有，則放入```ofproto.OFPP_FLODD```。

#### actions = [par…
在此先準備好一個 OFPActionOutput 的 actions，其對應的位址是剛剛經過判斷式設定的```out_port```。

#### if out\_port != ofpro…
判斷如果目的位止已經在 MAC 表中，則將建立此 Flow Entry（建立方式如判斷式成立後的執行動作）。值得注意的是，建立 Flow Entry 的優先權為1哦！

> add_flow 已經都出現完了，以下就先介紹它的用途。介紹完後，再回到 Packet-In。

## 新增 Flow Entry 的處理（add\_flow）

```python
def add_flow(self, datapath, priority, match, actions):
    ofproto = datapath.ofproto
    parser = datapath.ofproto_parser
 
    inst = [parser.OFPInstructionActions(ofproto.OFPIT_APPLY_ACTIONS,actions)]
 
    mod = parser.OFPFlowMod(datapath=datapath, priority=priority, match=match, instruction=inst)
    datapath.send_msg(mod)
```
#### inst = [parser.OFPInstruct…
用來定義封包所需要執行的動作。ofproto.OFPIT\_APPLY\_ACTIONS 對 Switch 來說，是指將指定規則對應的動作加入 Table 中，但如 Table 中已有同樣的規則（Match），則保留舊的動作，不覆寫。

> Instruction 是用來定義當封包滿足所規範的 Match 條件時，需要執行的動作。[參考來源](https://osrg.github.io/ryu-book/zh_tw/html/openflow_protocol.html?highlight=instruction)

#### mod = parser.OFPF…
產生一個```OFPFlowMod```類別（為將要傳出的資料類別）。```OFPFlowMod```可帶的參數很多，如想進一步了解，可以看看 Ryubook 中的補充（第12頁）。

#### datapath.send\_msg(mod)
將訊息送出。

> 接下來，回到Packet-In

```python
def _packet_in_handler(self, ev):
    #...
 
    data = None
    if msg.buffer_id == ofproto.OFP_NO_BUFFER:
        data = msg.data
 
    out = parser.OFPPacketOut(datapath=datapath, buffer_id=msg.buffer_id, in_port=in_port, actions=actions, data=data)
    datapath.send_msg(out)
```

#### if msg.buffer\_id == ofpr…
檢查一下 Buffer，然後把 msg.data 放入 data。

#### out = parser.OFPPacketOut(datapa…
將要傳送的類別，依需要的參數建立起來。

#### datapath.send\_msg(out)
最後，送出！

## 總結

雖然只是實現一個簡單 Switch 的功能，但裡面包含了許多 OpenFlow Switch 的基本知識。如對 OpenFlow Switch 的概念還不清楚，可以透過這隻 app，初步認識 OpenFlow Switch。

## 參考
[Ryubook](https://osrg.github.io/ryu-book/zh_tw/Ryubook.pdf)