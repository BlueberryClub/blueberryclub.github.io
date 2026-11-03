+++
date = '2026-10-21T14:00:00+08:00'
draft = true
title = 'SVG电力线路图绘制与渲染开发指南'
tags = ["SVG","电力拓扑","开发指南"]
series = ["使用教程"]
categories = ["前端", "SVG", "电力系统"]
+++

# SVG电力线路图绘制与渲染开发指南

## 1. 数据模型概述

电力线路图由两个文件组成：
- **XML文件**（CIM/RDF格式）：存储拓扑数据和设备属性
- **SVG文件**：存储图形渲染信息（图标定义、坐标、连线、文字）

### 1.1 XML 数据模型（IEC CIM）

XML 遵循 IEC CIM（公共信息模型）标准，核心元素：

```
Circuit（线路）
  └── CircuitSection（线段/干线/支线）
        └── Equipment（设备）
              ├── Disconnector（隔离开关）
              ├── Breaker（断路器）
              ├── LoadSwitch（负荷开关）
              ├── Fuse（熔断器）
              ├── Arrester（避雷器）
              ├── PoleSite（杆塔位置）
              ├── PowerTransformer（变压器）
              ├── FaultIndicator（故障指示器）
              ├── Junction（电缆终端头）
              └── ...
```

**关联关系通过 Terminal 建立：**

```xml
<!-- 设备：刀闸 -->
<cim:Disconnector rdf:ID="SWITCH_378010585067001">
    <cim:Naming.name>松岗线#1塔1T01刀闸</cim:Naming.name>
    <cim:PowerSystemResource.status>1</cim:PowerSystemResource.status>
    <cim:Switch.normalOpen>false</cim:Switch.normalOpen>
    <cim:PowerSystemResource.location>POINT(112.8696172 22.7891477)</cim:PowerSystemResource.location>
    <cim:PowerSystemResource.symbolRotation>231.08</cim:PowerSystemResource.symbolRotation>
    <cim:Equipment.MemberOf_EquipmentContainer rdf:resource="#CIRCUIT_378010547255901"/>
    <cim:Equipment.MemberOf_CircuitSection rdf:resource="#CIRSEC_378010547255911"/>
</cim:Disconnector>

<!-- 端子1：连接设备到节点A -->
<cim:Terminal rdf:ID="TERMINAL_378010585067001_1">
    <cim:Terminal.sequenceNumber>1</cim:Terminal.sequenceNumber>
    <cim:Terminal.ConductingEquipment rdf:resource="#SWITCH_378010585067001"/>
    <cim:Terminal.ConnectivityNode rdf:resource="#NODE_382220463449900"/>
</cim:Terminal>

<!-- 端子2：连接设备到节点B -->
<cim:Terminal rdf:ID="TERMINAL_378010585067001_2">
    <cim:Terminal.sequenceNumber>2</cim:Terminal.sequenceNumber>
    <cim:Terminal.ConductingEquipment rdf:resource="#SWITCH_378010585067001"/>
    <cim:Terminal.ConnectivityNode rdf:resource="#NODE_382220463450000"/>
</cim:Terminal>
```

**拓扑关系链：**
```
Equipment ←Terminal→ ConnectivityNode ←Terminal→ Equipment
```

### 1.2 SVG 图形模型

SVG 文件结构分为三层：

```
<svg>
  <defs>                    ← 第一层：通用图标定义（symbol）
  <g id="HeadClass">        ← 线路标题信息
  <g id="SubstationClass">  ← 变电站/配电房
  <g id="DisconnectorClass">← 按设备类型分层
  <g id="BreakerClass">
  <g id="ConnectiveLineClass"> ← 连接线（拓扑连线）
  <g id="ACLineSegmentClass">  ← 线路段（导线）
  <g id="TextClass">           ← 文字标注
  ...
</svg>
```

**完整的 Class 分层：**

| Class ID | 内容 | 说明 |
|----------|------|------|
| HeadClass | 线路名称、ID | 背景矩形 + 属性 |
| SubstationClass | 变电站/配电房 | 多边形框 + metadata |
| OtherClass | FTU/DTU等附属设备 | use引用 + metadata |
| ConnectivePointClass | 电缆终端头等连接点 | use引用 |
| DisconnectorClass | 隔离开关/刀闸 | use引用 |
| ArresterClass | 避雷器 | use引用 |
| PolePSRClass | 杆塔（铁塔/水泥杆） | use引用 |
| FaultIndicatorClass | 故障指示器 | use引用 |
| FuseClass | 熔断器 | use引用 |
| PowerTransformerClass | 变压器 | use引用 |
| BreakerClass | 断路器 | use引用 |
| LoadSwitchClass | 负荷开关 | use引用 |
| ConnectiveLineClass | 拓扑连接线 | path + CN_Ref |
| TextClass | 文字标注 | text元素 |
| ACLineSegmentClass | 导线段 | polyline |
| JointUseClass | 共管段 | polyline |
| BusbarSectionClass | 母线 | polyline |

---

## 2. 拓扑分析：从 XML 构建节点图

### 2.1 核心思路

从 XML 中提取三类数据：
1. **设备节点**（Equipment）：携带属性的图节点
2. **连接节点**（ConnectivityNode）：纯拓扑连接点
3. **边关系**（Terminal）：设备 ↔ 连接节点的映射

### 2.2 解析代码（Java）

```java
/**
 * XML 拓扑解析器
 * 从 CIM/RDF XML 中解析出设备节点和拓扑关系
 */
public class CimTopologyParser {

    // 命名空间映射
    private static final Map<String, String> NS = Map.of(
        "cim", "http://iec.ch/TC57/2003/CIM-schema-cim10#",
        "rdf", "http://www.w3.org/1999/02/22-rdf-syntax-ns#"
    );

    /**
     * 设备节点 —— 携带属性的图节点
     */
    @Data
    public static class EquipmentNode {
        private String id;           // rdf:ID，如 SWITCH_378010585067001
        private String type;         // 设备类型，如 Disconnector
        private String name;         // 设备名称
        private String mRID;         // 唯一标识
        private int status;          // 状态（1=运行）
        private String location;     // GPS坐标 POINT(lng lat)
        private double rotation;     // 图标旋转角度
        private boolean normalOpen;  // 常开（开关类设备）
        private String circuitId;    // 所属线路
        private String sectionId;    // 所属线段
        private List<String> terminalIds = new ArrayList<>(); // 关联端子
    }

    /**
     * 连接节点 —— 拓扑连接点，记录所有关联的端子
     * 一个 ConnectivityNode 可以连接多个 Terminal，
     * 形成 "设备A ←terminal→ 连接节点 ←terminal→ 设备B" 的拓扑关系
     */
    @Data
    public static class ConnectivityNode {
        private String id;    // NODE_xxx
        private String mRID;
        // 关联的所有端子（通过 Terminal.ConnectivityNode 反向索引）
        private List<String> terminalIds = new ArrayList<>();
    }

    /**
     * 获取该连接节点左侧的设备（端子序号=2 的设备，即出口方向）
     */
    public List<EquipmentNode> getLeftEquipments(TopologyGraph graph) {
        List<EquipmentNode> result = new ArrayList<>();
        for (String tid : terminalIds) {
            TerminalEdge t = graph.getTerminals().get(tid);
            if (t != null && t.getSequenceNumber() == 2) {
                EquipmentNode eq = graph.getEquipments().get(t.getEquipmentRef());
                if (eq != null) result.add(eq);
            }
        }
        return result;
    }

    /**
     * 获取该连接节点右侧的设备（端子序号=1 的设备，即入口方向）
     */
    public List<EquipmentNode> getRightEquipments(TopologyGraph graph) {
        List<EquipmentNode> result = new ArrayList<>();
        for (String tid : terminalIds) {
            TerminalEdge t = graph.getTerminals().get(tid);
            if (t != null && t.getSequenceNumber() == 1) {
                EquipmentNode eq = graph.getEquipments().get(t.getEquipmentRef());
                if (eq != null) result.add(eq);
            }
        }
        return result;
    }

    /**
     * 获取所有相邻设备（不区分左右）
     */
    public List<EquipmentNode> getAdjacentEquipments(TopologyGraph graph) {
        List<EquipmentNode> result = new ArrayList<>();
        for (String tid : terminalIds) {
            TerminalEdge t = graph.getTerminals().get(tid);
            if (t != null) {
                EquipmentNode eq = graph.getEquipments().get(t.getEquipmentRef());
                if (eq != null) result.add(eq);
            }
        }
        return result;
    }

    /**
     * 端子 —— 设备与连接节点的桥梁
     */
    @Data
    public static class TerminalEdge {
        private String id;
        private int sequenceNumber;       // 端子序号（1=入口，2=出口）
        private String equipmentRef;      // 关联设备 ID
        private String connectivityNodeRef; // 关联连接节点 ID
    }

    /**
     * 线路段 —— 干线/支线的树形结构
     */
    @Data
    public static class CircuitSection {
        private String id;
        private String name;         // 如"木厂支线"
        private String connectionKind; // 主干线 / 分支线
        private String parentId;     // 父线段ID（分支线才有）
        private String circuitId;    // 所属线路
    }

    /**
     * 解析结果
     */
    @Data
    public static class TopologyGraph {
        private Map<String, EquipmentNode> equipments = new LinkedHashMap<>();
        private Map<String, ConnectivityNode> nodes = new LinkedHashMap<>();
        private Map<String, TerminalEdge> terminals = new LinkedHashMap<>();
        private Map<String, CircuitSection> sections = new LinkedHashMap<>();
    }

    /**
     * 解析 XML 文件
     */
    public TopologyGraph parse(InputStream xmlInput) throws Exception {
        DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();
        factory.setNamespaceAware(true);
        Document doc = factory.newDocumentBuilder().parse(xmlInput);

        TopologyGraph graph = new TopologyGraph();

        // 1. 解析所有设备类型
        parseEquipments(doc, graph, "Disconnector");
        parseEquipments(doc, graph, "Breaker");
        parseEquipments(doc, graph, "LoadSwitch");
        parseEquipments(doc, graph, "Fuse");
        parseEquipments(doc, graph, "Arrester");
        parseEquipments(doc, graph, "PoleSite");
        parseEquipments(doc, graph, "PowerTransformer");
        parseEquipments(doc, graph, "FaultIndicator");
        parseEquipments(doc, graph, "Junction");

        // 2. 先解析连接节点（Terminal 反向索引依赖它）
        parseConnectivityNodes(doc, graph);

        // 3. 解析端子（同时建立 设备→端子 和 连接节点→端子 的反向关联）
        parseTerminals(doc, graph);

        // 4. 解析线路段树
        parseCircuitSections(doc, graph);

        return graph;
    }

    /**
     * 解析指定类型的设备
     */
    private void parseEquipments(Document doc, TopologyGraph graph, String cimType) {
        NodeList list = doc.getElementsByTagNameNS(NS.get("cim"), cimType);
        for (int i = 0; i < list.getLength(); i++) {
            Element elem = (Element) list.item(i);
            EquipmentNode node = new EquipmentNode();
            node.setId(elem.getAttributeNS(NS.get("rdf"), "ID"));
            node.setType(cimType);
            node.setName(getTagText(elem, NS.get("cim"), "Naming.name"));
            node.setMRID(getTagText(elem, NS.get("cim"), "Naming.mRID"));
            node.setStatus(parseInt(getTagText(elem, NS.get("cim"), "PowerSystemResource.status")));
            node.setLocation(getTagText(elem, NS.get("cim"), "PowerSystemResource.location"));
            node.setRotation(parseDouble(getTagText(elem, NS.get("cim"), "PowerSystemResource.symbolRotation")));

            // 开关类设备读取常开状态
            String normalOpen = getTagText(elem, NS.get("cim"), "Switch.normalOpen");
            if ("true".equals(normalOpen)) {
                node.setNormalOpen(true);
            }

            // 所属线路
            String circuitRef = getRefAttribute(elem, NS.get("cim"),
                "Equipment.MemberOf_EquipmentContainer");
            node.setCircuitId(stripRefPrefix(circuitRef));

            // 所属线段
            String sectionRef = getRefAttribute(elem, NS.get("cim"),
                "Equipment.MemberOf_CircuitSection");
            node.setSectionId(stripRefPrefix(sectionRef));

            graph.getEquipments().put(node.getId(), node);
        }
    }

    /**
     * 解析端子 —— 建立设备与连接节点的关联
     */
    private void parseTerminals(Document doc, TopologyGraph graph) {
        NodeList list = doc.getElementsByTagNameNS(NS.get("cim"), "Terminal");
        for (int i = 0; i < list.getLength(); i++) {
            Element elem = (Element) list.item(i);
            TerminalEdge edge = new TerminalEdge();
            edge.setId(elem.getAttributeNS(NS.get("rdf"), "ID"));
            edge.setSequenceNumber(parseInt(
                getTagText(elem, NS.get("cim"), "Terminal.sequenceNumber")));
            edge.setEquipmentRef(stripRefPrefix(
                getRefAttribute(elem, NS.get("cim"), "Terminal.ConductingEquipment")));
            edge.setConnectivityNodeRef(stripRefPrefix(
                getRefAttribute(elem, NS.get("cim"), "Terminal.ConnectivityNode")));

            graph.getTerminals().put(edge.getId(), edge);

            // 反向关联1：设备 → 端子列表
            EquipmentNode eq = graph.getEquipments().get(edge.getEquipmentRef());
            if (eq != null) {
                eq.getTerminalIds().add(edge.getId());
            }

            // 反向关联2：连接节点 → 端子列表（建立左右连接关系）
            ConnectivityNode cn = graph.getNodes().get(edge.getConnectivityNodeRef());
            if (cn != null) {
                cn.getTerminalIds().add(edge.getId());
            }
        }
    }

    /**
     * 解析连接节点
     */
    private void parseConnectivityNodes(Document doc, TopologyGraph graph) {
        NodeList list = doc.getElementsByTagNameNS(NS.get("cim"), "ConnectivityNode");
        for (int i = 0; i < list.getLength(); i++) {
            Element elem = (Element) list.item(i);
            ConnectivityNode cn = new ConnectivityNode();
            cn.setId(elem.getAttributeNS(NS.get("rdf"), "ID"));
            cn.setMRID(getTagText(elem, NS.get("cim"), "Naming.mRID"));
            graph.getNodes().put(cn.getId(), cn);
        }
    }

    /**
     * 解析线路段树
     */
    private void parseCircuitSections(Document doc, TopologyGraph graph) {
        NodeList list = doc.getElementsByTagNameNS(NS.get("cim"), "CircuitSection");
        for (int i = 0; i < list.getLength(); i++) {
            Element elem = (Element) list.item(i);
            CircuitSection sec = new CircuitSection();
            sec.setId(elem.getAttributeNS(NS.get("rdf"), "ID"));
            sec.setName(getTagText(elem, NS.get("cim"), "Naming.name"));
            sec.setConnectionKind(getTagText(elem, NS.get("cim"),
                "CircuitSection.connectionKind"));
            sec.setParentId(stripRefPrefix(
                getRefAttribute(elem, NS.get("cim"), "CircuitSection.Parent")));
            sec.setCircuitId(stripRefPrefix(
                getRefAttribute(elem, NS.get("cim"), "CircuitSection.MemberOf_Circuit")));
            graph.getSections().put(sec.getId(), sec);
        }
    }

    // ---- 工具方法 ----

    private String getTagText(Element parent, String ns, String localName) {
        NodeList list = parent.getElementsByTagNameNS(ns, localName);
        return list.getLength() > 0 ? list.item(0).getTextContent().trim() : "";
    }

    private String getRefAttribute(Element parent, String ns, String localName) {
        NodeList list = parent.getElementsByTagNameNS(ns, localName);
        if (list.getLength() == 0) return "";
        return ((Element) list.item(0)).getAttributeNS(
            "http://www.w3.org/1999/02/22-rdf-syntax-ns#", "resource");
    }

    private String stripRefPrefix(String ref) {
        // "#CIRCUIT_xxx" → "CIRCUIT_xxx"
        return ref.startsWith("#") ? ref.substring(1) : ref;
    }

    private int parseInt(String s) {
        try { return s.isEmpty() ? 0 : Integer.parseInt(s); }
        catch (NumberFormatException e) { return 0; }
    }

    private double parseDouble(String s) {
        try { return s.isEmpty() ? 0 : Double.parseDouble(s); }
        catch (NumberFormatException e) { return 0; }
    }
}
```

### 2.3 拓扑遍历：从出线开关沿线路走

```java
/**
 * 拓扑遍历器 —— 从出线开关开始，沿 ConnectivityNode 深度遍历整条线路
 */
public class TopologyTraverser {

    private final TopologyGraph graph;

    public TopologyTraverser(TopologyGraph graph) {
        this.graph = graph;
    }

    /**
     * 从指定设备开始，遍历所有连通设备
     * @param startEquipmentId 起始设备ID（通常是出线开关）
     * @param visitor 访问回调
     */
    public void traverse(String startEquipmentId, Consumer<EquipmentNode> visitor) {
        Set<String> visitedNodes = new HashSet<>();
        Set<String> visitedEquipments = new HashSet<>();
        EquipmentNode start = graph.getEquipments().get(startEquipmentId);
        if (start == null) return;
        doTraverse(start, visitedNodes, visitedEquipments, visitor);
    }

    private void doTraverse(EquipmentNode equipment,
                            Set<String> visitedNodes,
                            Set<String> visitedEquipments,
                            Consumer<EquipmentNode> visitor) {
        if (visitedEquipments.contains(equipment.getId())) return;
        visitedEquipments.add(equipment.getId());
        visitor.accept(equipment);

        // 从该设备的所有端子出发
        for (String terminalId : equipment.getTerminalIds()) {
            TerminalEdge terminal = graph.getTerminals().get(terminalId);
            if (terminal == null) continue;

            String nodeId = terminal.getConnectivityNodeRef();
            if (visitedNodes.contains(nodeId)) continue;
            visitedNodes.add(nodeId);

            // 从连接节点直接获取关联端子（已在解析阶段建立）
            ConnectivityNode cn = graph.getNodes().get(nodeId);
            if (cn == null) continue;

            for (String otherTid : cn.getTerminalIds()) {
                if (otherTid.equals(terminalId)) continue;
                TerminalEdge other = graph.getTerminals().get(otherTid);
                if (other == null) continue;
                EquipmentNode nextEquip = graph.getEquipments().get(other.getEquipmentRef());
                if (nextEquip != null) {
                    doTraverse(nextEquip, visitedNodes, visitedEquipments, visitor);
                }
            }
        }
    }

    /**
     * 获取指定线路下的所有设备，按线段分组
     */
    public Map<String, List<EquipmentNode>> groupBySection(String circuitId) {
        Map<String, List<EquipmentNode>> result = new LinkedHashMap<>();
        // 找到该线路的出线开关
        graph.getEquipments().values().stream()
            .filter(eq -> circuitId.equals(eq.getCircuitId()))
            .forEach(eq -> {
                result.computeIfAbsent(eq.getSectionId(), k -> new ArrayList<>())
                    .add(eq);
            });
        return result;
    }
}
```

---

## 3. SVG 渲染：图标、文字、连线

### 3.1 defs 中定义通用图标

SVG 的 `<defs>` 区域定义所有可复用的设备图标（symbol），通过 `id` 标识设备类型和状态：

**命名规则：** `{CIM类型}:{设备名称}@{状态码}`

| 状态码 | 含义 | 颜色 |
|--------|------|------|
| @0 | 合闸/正常 | rgb(0,255,0) 绿色 |
| @1 | 分闸 | rgb(255,0,0) 红色 |
| @4 | 接地/特殊 | rgb(255,0,255) 紫色 |

**图标定义示例：**

```xml
<defs>
    <!-- 通用端子（隐藏的连接点） -->
    <symbol id="terminal:端子" preserveAspectRatio="xMidYMid">
        <circle cy="0.0" r="1.0" stroke-width="1" cx="0.0" visibility="hidden"/>
    </symbol>

    <!-- 柱上隔离开关 - 合闸状态(绿色) -->
    <symbol viewBox="0 0 20.4 42.0"
            id="Disconnector:柱上隔离开关@0"
            preserveAspectRatio="xMidYMid">
        <!-- 上下两个端子连接点 -->
        <use x="10.2" y="1.0" terminal-index="0" xlink:href="#terminal:端子"/>
        <use x="10.2" y="41.0" terminal-index="1" xlink:href="#terminal:端子"/>
        <!-- 导线（绿色=合闸） -->
        <line x1="10.2" x2="10.2" stroke="rgb(0,255,0)"
              y1="27.5" y2="41.0" stroke-width="1"/>
        <line x1="10.2" x2="10.2" stroke="rgb(0,255,0)"
              y1="10.3" y2="1.0" stroke-width="1"/>
        <!-- 刀闸横杠 -->
        <line x1="7.2" x2="13.2" stroke="rgb(0,255,0)"
              y1="10.3" y2="10.3" stroke-width="1"/>
        <!-- 刀闸刀片（合闸=竖直） -->
        <line x1="10.0" x2="1.0" stroke="rgb(0,255,0)"
              y1="26.4" y2="9.4" stroke-width="1"/>
    </symbol>

    <!-- 柱上隔离开关 - 分闸状态(红色) -->
    <symbol viewBox="0 0 20.4 42.0"
            id="Disconnector:柱上隔离开关@1"
            preserveAspectRatio="xMidYMid">
        <use x="10.2" y="1.0" terminal-index="0" xlink:href="#terminal:端子"/>
        <use x="10.2" y="41.0" terminal-index="1" xlink:href="#terminal:端子"/>
        <line x1="10.2" x2="10.2" stroke="rgb(255,0,0)"
              y1="27.5" y2="41.0" stroke-width="1"/>
        <line x1="10.2" x2="10.2" stroke="rgb(255,0,0)"
              y1="10.3" y2="1.0" stroke-width="1"/>
        <line x1="7.2" x2="13.2" stroke="rgb(255,0,0)"
              y1="10.3" y2="10.3" stroke-width="1"/>
        <!-- 刀闸刀片（分闸=倾斜断开） -->
        <line x1="10.1" x2="5.9" stroke="rgb(255,0,0)"
              y1="26.2" y2="8.8" stroke-width="1"/>
    </symbol>
</defs>
```

### 3.2 在图层中引用图标

每个设备在对应的 Class 图层中用 `<use>` 引用 defs 中的 symbol：

```xml
<!-- DisconnectorClass 图层 -->
<g id="DisconnectorClass">
    <g id="380220621226700">
        <!-- 引用分闸状态的柱上隔离开关 -->
        <use x="832.5" y="1167.6"
             width="20.4" height="42.0"
             xlink:href="#Disconnector:柱上隔离开关@1"
             transform="rotate(90.0,842.7,1188.6)"/>
        <!-- 元数据：关联 XML 中的设备ID -->
        <metadata>
            <cge:PSR_Ref ObjectName="1T01"
                         ObjectID="SWITCH_378010585067001"/>
        </metadata>
    </g>
</g>
```

**关键属性说明：**

| 属性 | 作用 |
|------|------|
| `x`, `y` | 图标左上角在画布上的坐标 |
| `width`, `height` | 图标尺寸（对应 viewBox 的缩放） |
| `xlink:href` | 引用 defs 中的 symbol ID |
| `transform="rotate(angle,cx,cy)"` | 绕中心点旋转 |
| `transform="scale(sx,sy)"` | 缩放 |
| `metadata/cge:PSR_Ref/@ObjectID` | 关联 XML 设备 ID |

### 3.3 动态渲染代码（前端 JavaScript）

```javascript
/**
 * SVG 电力线路图渲染器
 */
class PowerLineRenderer {

    constructor(svgElement) {
        this.svg = svgElement;
        this.ns = 'http://www.w3.org/2000/svg';
        this.xlinkNs = 'http://www.w3.org/1999/xlink';
    }

    /**
     * 在 defs 中注册一个设备图标
     * @param {string} id - symbol ID，如 "Disconnector:柱上隔离开关@0"
     * @param {string} viewBox - 如 "0 0 20.4 42.0"
     * @param {Function} drawFn - 绘制函数，接收 SVGElement 参数
     */
    registerSymbol(id, viewBox, drawFn) {
        let defs = this.svg.querySelector('defs');
        if (!defs) {
            defs = document.createElementNS(this.ns, 'defs');
            this.svg.prepend(defs);
        }

        const symbol = document.createElementNS(this.ns, 'symbol');
        symbol.setAttribute('id', id);
        symbol.setAttribute('viewBox', viewBox);
        symbol.setAttribute('preserveAspectRatio', 'xMidYMid');
        drawFn(symbol);
        defs.appendChild(symbol);
    }

    /**
     * 注册柱上隔离开关图标
     */
    registerDisconnectorSymbol() {
        // 合闸（绿色）
        this.registerSymbol(
            'Disconnector:柱上隔离开关@0', '0 0 20.4 42.0',
            (symbol) => {
                this._addTerminal(symbol, 10.2, 1.0, 0);
                this._addTerminal(symbol, 10.2, 41.0, 1);
                this._addLine(symbol, 10.2, 27.5, 10.2, 41.0, 'rgb(0,255,0)');
                this._addLine(symbol, 10.2, 10.3, 10.2, 1.0, 'rgb(0,255,0)');
                this._addLine(symbol, 7.2, 10.3, 13.2, 10.3, 'rgb(0,255,0)');
                this._addLine(symbol, 10.0, 26.4, 1.0, 9.4, 'rgb(0,255,0)');
            }
        );
        // 分闸（红色）
        this.registerSymbol(
            'Disconnector:柱上隔离开关@1', '0 0 20.4 42.0',
            (symbol) => {
                this._addTerminal(symbol, 10.2, 1.0, 0);
                this._addTerminal(symbol, 10.2, 41.0, 1);
                this._addLine(symbol, 10.2, 27.5, 10.2, 41.0, 'rgb(255,0,0)');
                this._addLine(symbol, 10.2, 10.3, 10.2, 1.0, 'rgb(255,0,0)');
                this._addLine(symbol, 7.2, 10.3, 13.2, 10.3, 'rgb(255,0,0)');
                this._addLine(symbol, 10.1, 26.2, 5.9, 8.8, 'rgb(255,0,0)');
            }
        );
    }

    /**
     * 渲染一个设备到指定图层
     */
    renderEquipment(layerId, equipmentId, symbolId, x, y, width, height,
                    rotation, objectName, objectId) {
        let layer = this.svg.querySelector(`#${layerId}`);
        if (!layer) {
            layer = document.createElementNS(this.ns, 'g');
            layer.setAttribute('id', layerId);
            this.svg.appendChild(layer);
        }

        const g = document.createElementNS(this.ns, 'g');
        g.setAttribute('id', equipmentId);

        // use 引用 symbol
        const use = document.createElementNS(this.ns, 'use');
        use.setAttribute('x', x);
        use.setAttribute('y', y);
        use.setAttribute('width', width);
        use.setAttribute('height', height);
        use.setAttributeNS(this.xlinkNs, 'xlink:href', `#${symbolId}`);
        if (rotation) {
            const cx = x + width / 2;
            const cy = y + height / 2;
            use.setAttribute('transform', `rotate(${rotation},${cx},${cy})`);
        }
        g.appendChild(use);

        // metadata 关联
        const metadata = document.createElementNS(this.ns, 'metadata');
        const psrRef = document.createElementNS(
            'http://iec.ch/TC57/2005/SVG-schema#', 'cge:PSR_Ref');
        psrRef.setAttribute('ObjectName', objectName || '');
        psrRef.setAttribute('ObjectID', objectId || '');
        metadata.appendChild(psrRef);
        g.appendChild(metadata);

        layer.appendChild(g);
    }

    /**
     * 渲染文字标注
     */
    renderText(layerId, id, x, y, text, options = {}) {
        let layer = this.svg.querySelector(`#${layerId}`);
        if (!layer) {
            layer = document.createElementNS(this.ns, 'g');
            layer.setAttribute('id', layer);
            this.svg.appendChild(layer);
        }

        const g = document.createElementNS(this.ns, 'g');
        g.setAttribute('id', id);

        const textEl = document.createElementNS(this.ns, 'text');
        textEl.setAttribute('x', x);
        textEl.setAttribute('y', y);
        textEl.setAttribute('text-anchor', options.anchor || 'middle');
        textEl.setAttribute('font-size', options.fontSize || '14');
        textEl.setAttribute('fill', options.fill || 'rgb(0,0,0)');
        textEl.setAttribute('font-family', options.fontFamily || 'KaiTi');
        if (options.linkLayer) {
            textEl.setAttribute('link-layer', options.linkLayer);
        }
        textEl.textContent = text;
        g.appendChild(textEl);
        layer.appendChild(g);
    }

    /**
     * 渲染导线段
     */
    renderLineSegment(layerId, id, points, options = {}) {
        let layer = this.svg.querySelector(`#${layerId}`);
        if (!layer) {
            layer = document.createElementNS(this.ns, 'g');
            layer.setAttribute('id', layer);
            this.svg.appendChild(layer);
        }

        const g = document.createElementNS(this.ns, 'g');
        g.setAttribute('id', id);

        const polyline = document.createElementNS(this.ns, 'polyline');
        polyline.setAttribute('fill', 'none');
        polyline.setAttribute('stroke', options.stroke || 'rgb(0,0,0)');
        polyline.setAttribute('stroke-width', options.strokeWidth || '1');
        polyline.setAttribute('points', points); // "x1,y1 x2,y2 x3,y3"
        if (options.dashArray) {
            polyline.setAttribute('stroke-dasharray', options.dashArray);
        }
        g.appendChild(polyline);
        layer.appendChild(g);
    }

    // ---- 内部工具方法 ----

    _addTerminal(symbol, x, y, index) {
        const use = document.createElementNS(this.ns, 'use');
        use.setAttribute('x', x);
        use.setAttribute('y', y);
        use.setAttribute('terminal-index', index);
        use.setAttributeNS(this.xlinkNs, 'xlink:href', '#terminal:端子');
        symbol.appendChild(use);
    }

    _addLine(symbol, x1, y1, x2, y2, color) {
        const line = document.createElementNS(this.ns, 'line');
        line.setAttribute('x1', x1);
        line.setAttribute('y1', y1);
        line.setAttribute('x2', x2);
        line.setAttribute('y2', y2);
        line.setAttribute('stroke', color);
        line.setAttribute('stroke-width', '1');
        symbol.appendChild(line);
    }
}
```

---

## 4. 线路颜色渲染

### 4.1 颜色配置方案

```javascript
/**
 * 线路颜色配置
 */
const LINE_COLOR_CONFIG = {
    // 按电压等级
    voltage: {
        '110kV':  { stroke: 'rgb(255,0,0)',   label: '110千伏' },
        '10kV':   { stroke: 'rgb(0,128,0)',    label: '10千伏' },
        '380V':   { stroke: 'rgb(0,0,255)',    label: '380伏' },
    },

    // 按线路状态
    status: {
        running:    { stroke: 'rgb(0,255,0)',   label: '运行' },
        stopped:    { stroke: 'rgb(255,0,0)',    label: '停运' },
        fault:      { stroke: 'rgb(255,0,255)',  label: '故障' },
        maintenance:{ stroke: 'rgb(255,165,0)',  label: '检修' },
    },

    // 按线段类型
    section: {
        trunk:    { stroke: 'rgb(0,0,0)',     strokeWidth: '2', label: '主干线' },
        branch:   { stroke: 'rgb(0,0,0)',     strokeWidth: '1', label: '分支线' },
        connective:{ stroke: 'rgb(0,0,0)',    strokeWidth: '1', label: '联络线' },
    },

    // 按相别（A/B/C相）
    phase: {
        A: { stroke: 'rgb(255,0,0)',     label: 'A相' },
        B: { stroke: 'rgb(0,255,0)',     label: 'B相' },
        C: { stroke: 'rgb(0,0,255)',     label: 'C相' },
    },

    // 按馈线（不同馈线不同颜色）
    feeder: [
        'rgb(0,128,0)',     // 馈线1 绿色
        'rgb(0,0,255)',     // 馈线2 蓝色
        'rgb(255,128,0)',   // 馈线3 橙色
        'rgb(128,0,255)',   // 馈线4 紫色
        'rgb(0,128,128)',   // 馈线5 青色
        'rgb(255,0,128)',   // 馈线6 品红
    ],
};
```

### 4.2 按馈线着色

```javascript
/**
 * 按馈线对线路进行颜色渲染
 */
class LineColorRenderer {

    constructor(svgElement, colorConfig) {
        this.svg = svgElement;
        this.config = colorConfig || LINE_COLOR_CONFIG;
    }

    /**
     * 给指定馈线的所有导线段着色
     * @param {string} circuitId - 线路ID
     * @param {string} color - 颜色，如 "rgb(255,0,0)"
     */
    colorCircuit(circuitId, color) {
        // 找到 ACLineSegmentClass 下所有属于该线路的线段
        const layer = this.svg.querySelector('#ACLineSegmentClass');
        if (!layer) return;

        const segments = layer.querySelectorAll('g');
        segments.forEach(g => {
            const metadata = g.querySelector('cge\\:PSR_Ref, PSR_Ref');
            if (!metadata) return;
            const objectId = metadata.getAttribute('ObjectID') || '';
            // SEG_xxx 属于该线路则着色
            if (this._segmentBelongsToCircuit(objectId, circuitId)) {
                const polyline = g.querySelector('polyline');
                if (polyline) {
                    polyline.setAttribute('stroke', color);
                }
            }
        });
    }

    /**
     * 给指定状态的设备着色
     */
    colorEquipmentByStatus(equipmentId, status) {
        const colorMap = this.config.status;
        const color = colorMap[status]?.stroke || 'rgb(0,0,0)';

        // 查找设备所在图层
        const allLayers = this.svg.querySelectorAll('[id$="Class"]');
        for (const layer of allLayers) {
            const g = layer.querySelector(`#${equipmentId}`);
            if (!g) continue;

            const use = g.querySelector('use');
            if (!use) continue;

            // 切换 symbol 引用（@0=合闸, @1=分闸）
            const href = use.getAttributeNS(
                'http://www.w3.org/1999/xlink', 'xlink:href');
            if (href) {
                const baseId = href.replace(/@\d+$/, '');
                const statusCode = status === 'running' ? '0' : '1';
                use.setAttributeNS('http://www.w3.org/1999/xlink',
                    'xlink:href', `${baseId}@${statusCode}`);
            }
        }
    }

    /**
     * 高亮指定线段（主干线/分支线）
     */
    highlightSection(sectionId, color, strokeWidth) {
        const layer = this.svg.querySelector('#ACLineSegmentClass');
        if (!layer) return;

        // 通过 metadata 中的 ObjectID 匹配
        layer.querySelectorAll('g').forEach(g => {
            const ref = g.querySelector('cge\\:PSR_Ref, PSR_Ref');
            if (ref && ref.getAttribute('ObjectID')?.includes(sectionId)) {
                const polyline = g.querySelector('polyline');
                if (polyline) {
                    polyline.setAttribute('stroke', color);
                    polyline.setAttribute('stroke-width', strokeWidth || '3');
                }
            }
        });
    }

    /**
     * 清除所有自定义着色，恢复原始颜色
     */
    resetColors() {
        this.svg.querySelectorAll('polyline').forEach(p => {
            p.removeAttribute('stroke');
            p.removeAttribute('stroke-width');
        });
    }

    _segmentBelongsToCircuit(segmentId, circuitId) {
        // 通过拓扑分析结果判断线段归属
        // 实际项目中从后端接口获取映射关系
        return segmentId.includes(circuitId.replace('CIRCUIT_', ''));
    }
}
```

### 4.3 使用示例

```javascript
// 初始化渲染器
const svgEl = document.getElementById('power-line-svg');
const renderer = new PowerLineRenderer(svgEl);
const colorRenderer = new LineColorRenderer(svgEl);

// 注册图标
renderer.registerDisconnectorSymbol();

// 渲染设备
renderer.renderEquipment(
    'DisconnectorClass',       // 图层
    '380220621226700',         // 设备图形ID
    'Disconnector:柱上隔离开关@1', // symbol引用（分闸状态）
    832.5, 1167.6,             // 坐标
    20.4, 42.0,                // 尺寸
    90.0,                      // 旋转角度
    '1T01',                    // 显示名称
    'SWITCH_378010585067001'   // XML中的设备ID
);

// 渲染文字
renderer.renderText(
    'TextClass',
    '380220621227200',
    848.8, 1210.0,
    '1T01',
    { anchor: 'middle', fontSize: '14', linkLayer: 'DisconnectorClass' }
);

// 渲染导线段
renderer.renderLineSegment(
    'ACLineSegmentClass',
    '380220621229600',
    '882.7,1188.6 934.7,1188.6',
    { stroke: 'rgb(0,0,0)', strokeWidth: '1' }
);

// 按馈线着色
colorRenderer.colorCircuit('CIRCUIT_378010547255901', 'rgb(0,128,0)');

// 按状态着色
colorRenderer.colorEquipmentByStatus('SWITCH_378010585067001', 'running');
```

---

## 5. XML 与 SVG 的关联映射

两个文件通过 ID 关联：

```
XML                              SVG
─────────────────────────────────────────────────────
cim:Disconnector@rdf:ID  ←→  cge:PSR_Ref@ObjectID
  "SWITCH_378010585067001"      "SWITCH_378010585067001"

cim:Terminal.ConnectivityNode ←→ cge:CN_Ref@LinkObjectID
  "NODE_382220463450000"         "380220621226700@1"

cim:CircuitSection@rdf:ID  ←→  TextClass text@link-layer
  "CIRSEC_378010547255911"       "ACLineSegmentClass"
```

**映射代码：**

```java
/**
 * XML-SVG 关联映射器
 */
public class CimSvgMapper {

    /**
     * 从 SVG metadata 中提取 ObjectID，与 XML 设备关联
     */
    public Map<String, EquipmentNode> mapSvgToXml(
            TopologyGraph graph, Document svgDoc) {

        Map<String, EquipmentNode> mapping = new LinkedHashMap<>();

        // 遍历 SVG 中所有 cge:PSR_Ref 元素
        NodeList psrRefs = svgDoc.getElementsByTagNameNS(
            "http://iec.ch/TC57/2005/SVG-schema#", "PSR_Ref");

        for (int i = 0; i < psrRefs.getLength(); i++) {
            Element ref = (Element) psrRefs.item(i);
            String objectId = ref.getAttribute("ObjectID");
            if (objectId == null || objectId.isEmpty()) continue;

            EquipmentNode eq = graph.getEquipments().get(objectId);
            if (eq != null) {
                mapping.put(objectId, eq);
            }
        }
        return mapping;
    }
}
```

---

## 6. 完整处理流程

```
1. 加载 XML → CimTopologyParser.parse()
   ├── 解析 Equipment 节点（带属性）
   ├── 解析 Terminal 边（建立关联）
   ├── 解析 ConnectivityNode（连接点）
   └── 解析 CircuitSection（线段树）

2. 拓扑遍历 → TopologyTraverser.traverse()
   ├── 从出线开关开始 BFS/DFS
   ├── 按 ConnectivityNode 跳转
   └── 输出有序设备列表

3. 加载 SVG → 解析 defs 中的 symbol 定义
   ├── 提取所有 symbol id（设备类型+状态）
   ├── 提取 viewBox 和图形元素
   └── 缓存到内存

4. 渲染 → PowerLineRenderer
   ├── 按设备类型分层渲染（use引用symbol）
   ├── 渲染文字标注（TextClass）
   └── 渲染导线段（ACLineSegmentClass）

5. 着色 → LineColorRenderer
   ├── 按馈线着色
   ├── 按状态着色
   └── 按线段类型着色
```
