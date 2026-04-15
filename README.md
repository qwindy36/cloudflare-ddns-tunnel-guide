# Cloudflare动态接入与隧道建站指南
一份面向Windows服务器的实战指南，介绍如何使用Cloudflare DNS、动态IPv6更新与Tunnel实现稳定远程接入和多站点发布。
## 基于Cloudflare DNS与Tunnel的服务器接入与多站点发布方案

本方案面向一台公网地址不稳定、同时承载多个本地服务的Windows服务器，目标是在保证远程管理可达性的基础上，实现多站点的统一域名发布与简化运维。整体上，本方案经历了两个连续演进阶段：首先，通过Cloudflare DNS中的AAAA记录与定时更新脚本，构建基于动态IPv6的服务器远程接入机制；随后，在服务器端常驻部署`cloudflared`，引入Cloudflare Tunnel，将不同子域名映射到服务器内部不同本地端口，从而形成“远程管理入口+多站点发布入口”并存的分层接入架构。Cloudflare官方文档表明，DNS记录的代理状态决定了请求是直接解析到源站IP，还是经由Cloudflare代理层处理；同时，Tunnel支持将公网hostname映射到本地服务，并由常驻的`cloudflared`负责维持到Cloudflare的连接。([Cloudflare Docs](https://developers.cloudflare.com/dns/proxy-status/?utm_source=chatgpt.com))

### 1. 第一阶段：基于AAAA记录的动态IPv6远程接入

方案初期的核心需求是通过固定域名稳定连接服务器，而不是优先发布Web站点。由于服务器公网IPv6地址可能发生变化，因此在Cloudflare DNS中为指定主机名配置AAAA记录，并通过脚本周期性获取服务器最新IPv6地址，再调用Cloudflare DNS API对该AAAA记录进行更新。这样，虽然服务器实际公网地址是动态变化的，但对外提供的访问入口始终保持为同一域名，从而实现一种轻量级的动态DNS机制。Cloudflare官方API支持对既有DNS记录进行编辑更新，这使得该自动同步链路具备可实现性。([Cloudflare Docs](https://developers.cloudflare.com/api/resources/dns/subresources/records/methods/edit/?utm_source=chatgpt.com))

在这一阶段，SSH访问链路本质上是“域名解析到服务器当前IPv6，再由客户端直接连接服务器22端口”的传统直连模式。需要特别说明的是，Cloudflare文档明确指出，可被代理的A、AAAA和CNAME记录仅适用于承载HTTP或HTTPS流量的场景，因此原生SSH这类非HTTP服务并不适合依赖常规橙云代理进行透传。在该前提下，SSH使用的AAAA记录应当保持为DNS only状态，使客户端直接获取服务器真实IPv6地址并发起TCP连接。换言之，在该阶段中，Cloudflare承担的是权威DNS与动态解析托管角色，而不是SSH会话的中继节点。([Cloudflare Docs](https://developers.cloudflare.com/dns/proxy-status/limitations/?utm_source=chatgpt.com))

### 2. 第二阶段：引入Cloudflare Tunnel实现多站点统一发布

在解决“如何稳定找到服务器”这一基础问题之后，方案进一步演进到“如何统一管理服务器上的多个站点和本地服务”。为此，在Windows服务器上部署并常驻运行`cloudflared`，使服务器主动向Cloudflare建立持续的出站连接。Cloudflare官方文档指出，Tunnel的发布模型是将公网hostname映射到本地service，并建议大多数场景使用远程托管的Tunnel；在Windows环境下，`cloudflared`可以通过系统服务方式安装并长期运行。与此同时，若服务器处于较严格的网络环境下，Cloudflare要求其能够出站访问相应端口，例如`7844`，以保证Tunnel连接建立和维持。([Cloudflare Docs](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/do-more-with-tunnels/local-management/?utm_source=chatgpt.com))

在本方案中，`cloudflared`一旦作为某个Tunnel的connector稳定在线，后续新增站点时通常不再需要在服务器端重复执行额外的“绑定”操作，而是直接在Cloudflare控制台中为该Tunnel新增Published application route，将某个子域名映射到服务器本地指定端口即可。例如，可以将`docs.example.com`映射到`https://localhost:8080`，也可以将其他子域名分别映射到`http://localhost:3000`、`http://localhost:5000`等不同本地服务。Cloudflare官方对这一过程的定义非常明确：Tunnel route负责把公网hostname映射到本地service；在完整DNS托管场景下，添加Published application时还可以自动创建对应DNS记录。([Cloudflare Docs](https://developers.cloudflare.com/tunnel/setup/?utm_source=chatgpt.com))

需要强调的是，这里的“自动”并不是指Cloudflare会自动扫描服务器上所有已开启端口并完成发布，而是指：在`cloudflared`已经常驻且Tunnel已经连通的前提下，新增一条“子域名→本地端口”的发布规则后，在线的connector会自动接收Cloudflare侧下发的路由配置，并将符合该hostname的流量转发到指定本地服务。因此，后续新增站点的主要操作重心转移到了Cloudflare控制台，而服务器端只需确保相应本地服务已经启动、监听端口正确、协议填写一致即可。若本地服务未启动、端口填写错误或HTTP/HTTPS协议不匹配，Cloudflare官方排障文档中对应的常见表现即为连接拒绝、502等回源失败问题。([Cloudflare Docs](https://developers.cloudflare.com/tunnel/setup/?utm_source=chatgpt.com))

### 3. 整体技术路径与系统架构

经过上述两个阶段的演进，系统最终形成了“DNS动态解析路径”与“Tunnel应用发布路径”并存的双通道架构。其中，远程管理入口主要沿用基于AAAA记录的动态IPv6直连机制，即客户端访问SSH专用子域名后，经Cloudflare DNS解析获得服务器最新IPv6地址，再直接连接服务器开放的SSH端口；而Web站点、面板、接口服务等应用入口则通过Cloudflare Tunnel发布，即外部请求先到达Cloudflare边缘节点，再由Cloudflare按照Published application route将流量转发到服务器上常驻的`cloudflared`，最终落到对应的`localhost:端口`服务。前者解决的是“如何定位并直连这台服务器”，后者解决的是“如何对服务器内部多个服务进行统一域名管理和外部发布”。([Cloudflare Docs](https://developers.cloudflare.com/dns/proxy-status/?utm_source=chatgpt.com))

从访问链路角度看，本方案可概括为两条不同的数据通路。其一为SSH管理链路：`管理域名→Cloudflare DNS AAAA解析→服务器当前IPv6→SSH直连22端口`；其二为站点发布链路：`业务子域名→Cloudflare边缘节点→Tunnel路由→服务器端cloudflared→localhost对应端口`。这意味着，同一域名体系下的不同子域名可以承担不同职责：一部分用于服务器级远程运维入口，另一部分用于应用级服务发布入口，从而实现命名统一而接入方式分层。([Cloudflare Docs](https://developers.cloudflare.com/dns/proxy-status/limitations/?utm_source=chatgpt.com))

### 4. 方案特点与工程意义

该方案的显著特点在于，它并未简单地以前一种技术替代后一种技术，而是形成了递进式、叠加式的技术体系。动态AAAA方案优先解决了公网IPv6变化导致的远程连接不稳定问题，使服务器获得了一个逻辑固定、物理可变的访问入口；在此基础上，再引入Tunnel实现多站点的统一发布与集中管理，从而避免每新增一个站点都需要重新暴露新的公网端口或依赖新的公网IP配置。Cloudflare官方文档说明，远程托管Tunnel的配置保存在Cloudflare侧，可通过Dashboard、API或Terraform进行统一管理，这使得后续站点扩展、子域名调整以及服务迁移都具有更高的灵活性。([Cloudflare Docs](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/do-more-with-tunnels/local-management/?utm_source=chatgpt.com))

从安全与运维角度看，该架构还具备较强的工程实用性。对于Web类服务，借助Tunnel可以将外部访问入口前移到Cloudflare边缘，而不必继续依赖服务器直接暴露公网应用端口；Cloudflare关于Tunnel与防火墙的说明也表明，在适当配置下，源站可以收紧入站策略，仅保留必要出站能力。与之相对，SSH入口仍然保持独立的动态DNS直连路径，便于基础运维、故障恢复以及不依赖Web发布链路的服务器管理。这种“管理面与业务面分离”的架构设计，使系统在可达性、扩展性与可维护性之间取得了较好的平衡。([Cloudflare Docs](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/configure-tunnels/tunnel-with-firewall/?utm_source=chatgpt.com))

### 5. 小结

综上，本方案的完整技术路径可以概括为：首先，基于Cloudflare DNS中的AAAA记录与定时更新脚本，实现服务器公网IPv6变化场景下的稳定域名接入；随后，在Windows服务器上常驻部署`cloudflared`并接入Cloudflare Tunnel，将不同业务子域名映射到服务器内部不同本地端口，从而实现多站点的集中发布与统一管理。最终形成的系统架构中，SSH等基础管理服务继续沿用动态DNS直连路径，Web站点及其他本地应用则通过Tunnel发布路径对外提供服务，两者共同构成了兼顾远程运维与业务发布的分层式接入体系。([Cloudflare Docs](https://developers.cloudflare.com/api/resources/dns/subresources/records/methods/edit/?utm_source=chatgpt.com))

------

## 6. 部署步骤说明

本节用于说明整套方案的实际部署流程。结合当前使用方式，整体部署可分为两个部分：一是完成基于Cloudflare DNS的动态IPv6远程接入；二是在Windows服务器上部署并常驻`cloudflared`，通过Tunnel发布多个本地站点。为保证后续扩展方便，以下步骤默认采用“**一个域名统一托管在Cloudflare，服务器端常驻一个可在线的Tunnel connector，新增站点时主要在Cloudflare控制台完成子域名到本地端口的映射**”这一思路。

### 6.1 前置条件

在开始部署前，需要先满足以下基本条件。其一，目标域名已经成功接入Cloudflare，且域名解析权已经切换到Cloudflare。其二，Windows服务器能够正常联网，并且能够出站访问Cloudflare。其三，服务器上已经具备本地服务运行环境，例如IIS、Nginx、Node.js、Python Web服务或其他监听本地端口的程序。其四，已经明确规划好各子域名对应的用途，例如哪个子域名用于SSH管理入口，哪个子域名用于博客、面板、接口服务或其他站点。

在整个方案中，建议一开始就把子域名职责区分清楚。例如，可将一个专用子域名保留给服务器远程管理，使用AAAA动态解析方式接入；而将其他业务类子域名统一纳入Tunnel发布体系。这样后续维护时，不同入口的用途和链路会更加清晰。

### 6.2 第一部分：部署动态IPv6远程接入

第一部分的目标是解决“服务器公网IPv6会变化，但希望始终通过固定域名SSH连接服务器”的问题。具体做法是在Cloudflare DNS中创建一个用于远程管理的AAAA记录，并通过脚本定时将其更新为服务器当前最新IPv6地址。

首先，在Cloudflare控制台中进入目标域名的DNS管理界面，新建一个AAAA记录。该记录对应一个专门用于远程管理的主机名，例如`ssh.example.com`。记录值初始填写服务器当前公网IPv6地址。这里要注意，这条记录应作为“**直连入口**”使用，因此应保持为**仅DNS解析**状态，而不应依赖普通站点代理逻辑。

完成AAAA记录创建后，在Windows服务器上编写或部署一个定时更新脚本。这个脚本的核心逻辑包括三个步骤：首先读取服务器当前的公网IPv6地址；然后读取Cloudflare中该AAAA记录当前保存的IP值；最后在两者不一致时，通过Cloudflare API将AAAA记录更新为最新地址。为了降低无意义请求频率，可以在脚本中增加“只有IP变化时才更新”的判断逻辑，并将运行结果写入日志文件，便于后续排查。

接着，在Windows任务计划程序中为该脚本创建定时任务。建议设置为开机自动启用，并按照固定间隔执行，例如每5分钟、10分钟或15分钟检查一次。这样一来，即便公网IPv6地址变化，Cloudflare DNS中的AAAA记录也会在较短时间内被同步更新，从而保证域名访问始终指向最新的服务器地址。

在这一步部署完成后，可以通过域名方式远程管理服务器。此时整个SSH访问链路为：客户端发起对管理域名的解析请求，经Cloudflare DNS返回当前AAAA记录值，再由客户端直接连接该IPv6地址上的SSH服务端口。至此，服务器的基础远程接入能力已经建立完成。

### 6.3 第二部分：在Windows服务器上部署cloudflared

第二部分的目标是让服务器具备通过Cloudflare Tunnel发布本地站点的能力。这里的核心不是再次暴露公网端口，而是让服务器上的`cloudflared`长期在线，作为本机和Cloudflare之间的常驻连接器。

首先，在Windows服务器上下载安装`cloudflared`。安装完成后，可在命令行中检查程序是否可正常运行，并确认版本信息正常显示。建议将其放置在固定目录中，避免后续因路径变化导致服务失效。

然后，在Cloudflare控制台中创建一个Tunnel，并获取对应的连接令牌或接入方式。当前使用思路中，推荐采用远程托管方式，即Tunnel的主要路由配置保存在Cloudflare侧，而服务器端只负责运行`cloudflared`并维持连接。这样后续新增站点时，不需要频繁回到服务器本地修改配置文件，管理上更加简洁。

创建Tunnel后，在Windows服务器上执行对应命令，将当前这台服务器接入该Tunnel。完成接入后，需要进一步将`cloudflared`安装为Windows系统服务，使其在系统启动后自动运行。这样，服务器每次重启后都能够自动恢复与Cloudflare的连接，而不需要手动再次启动。

服务安装完成后，应立即检查其运行状态，确认`cloudflared`已经处于在线状态。此时可以在Cloudflare控制台中查看Tunnel状态是否显示为“健康”或“已连接”。只有当这一步成功后，后续子域名到本地端口的映射配置才能真正生效。

### 6.4 第三部分：通过Tunnel发布第一个站点

当`cloudflared`已经在Windows服务器上常驻并在线后，就可以开始发布本地站点。这里的核心思路是：**先让本地服务在服务器上运行并监听一个本地端口，再在Cloudflare控制台中将某个子域名映射到该本地端口。**

以发布一个Web站点为例，首先需要在Windows服务器上启动对应服务，并确认它已经监听在某个端口，例如`localhost:3000`、`localhost:8080`或`localhost:5000`。在正式绑定前，建议先在服务器本机浏览器中测试`http://localhost:端口`或`https://localhost:端口`能否正常访问，确保应用本身已经运行正常。

之后，进入Cloudflare控制台中的Tunnel管理页面，在当前Tunnel下新增一个Published application。填写时，需要指定三个关键信息：一是子域名，例如`blog.example.com`；二是协议类型，例如HTTP、HTTPS或其他支持的服务类型；三是对应的本地服务地址，例如`http://localhost:3000`。保存后，Cloudflare就会建立一条“公网子域名→本地服务端口”的转发关系。

在这一机制下，外部用户访问该子域名时，请求会先到达Cloudflare边缘节点，再通过Tunnel送到Windows服务器上的`cloudflared`，最后由`cloudflared`转发给本地的3000端口服务。这样，一个新站点就发布完成了。

### 6.5 第四部分：后续新增站点的扩展方式

在当前方案下，`cloudflared`只要已经在服务器端常驻成功，后续新增站点的工作量会明显下降。此时通常不再需要在服务器上重复执行任何“绑定Cloudflare”的操作，也不需要每次新建一个站点都重新安装或重新配置`cloudflared`。后续扩展的基本流程可以概括为：**先在服务器本地启动新服务，再在Cloudflare控制台中添加一条新的子域名到本地端口的映射。**

例如，如果后续又新建了一个后台管理面板，并让它监听在`localhost:9090`，那么只需要在Tunnel中再增加一条新规则，例如将`admin.example.com`映射到`http://localhost:9090`。如果再新建一个接口服务监听在`localhost:7001`，则可再增加一个子域名，将其映射到对应端口。这样，同一台Windows服务器上的多个本地服务就都可以通过不同子域名对外发布。

不过需要注意的是，这里的“方便扩展”并不意味着Cloudflare会自动识别服务器上有哪些新端口可用。新增站点时，仍然需要人为明确指定“哪个子域名对应哪个本地端口、使用什么协议”。真正自动化的是：一旦Cloudflare侧保存了新的映射规则，已经在线的`cloudflared`会自动接收这些配置，不需要再手动回服务器重新执行一遍绑定过程。

### 6.6 第五部分：日常运维检查要点

为了保证整套架构长期稳定运行，建议在日常维护中重点关注以下几个方面。

首先，要定期检查动态IPv6更新脚本是否正常运行。可以通过查看任务计划程序执行记录、脚本日志以及Cloudflare DNS中的AAAA记录值，确认当前管理域名仍然指向服务器最新公网IPv6地址。如果这里失效，就会直接影响SSH远程连接。

其次，要定期检查`cloudflared`服务是否在线。若Tunnel状态异常，虽然本地站点本身可能仍在运行，但外部用户将无法通过对应子域名访问这些服务。因此，服务器重启、系统更新或网络异常后，应优先确认`cloudflared`服务是否自动恢复运行。

再次，要检查每个站点的本地服务是否仍然按预期监听对应端口。因为Tunnel只负责转发，不负责启动业务程序。如果本地应用崩溃、端口变更或协议切换，而Cloudflare侧映射规则没有同步更新，就会出现外部访问失败的问题。

最后，建议对所有子域名建立一份对应关系清单，记录其用途、接入方式和目标端口。例如，应明确标注哪些子域名属于AAAA动态直连类，哪些属于Tunnel发布类，各自指向什么服务、使用什么协议。这样在后续扩容、迁移或故障排查时，能够快速定位问题所在。

### 6.7 部署结果总结

完成上述部署后，整套系统将形成如下运行格局：服务器管理入口通过Cloudflare DNS中的AAAA记录与定时更新脚本维持稳定可达，负责远程SSH接入；业务站点入口则通过Windows服务器上常驻的`cloudflared`与Cloudflare Tunnel实现统一发布，不同子域名分别映射到服务器内部不同本地端口。这样既保留了基础运维的独立通道，又实现了多站点的集中托管与便捷扩展，能够满足后续持续增加站点和服务的需求。



