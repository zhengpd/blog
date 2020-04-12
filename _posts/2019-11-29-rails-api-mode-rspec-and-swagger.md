# Rails API Mode, RSpec and Swagger

ZHENG Piao-Dan @ 2019-11-29

## 零

我期望所有我用到的第三方代码都有详细的文档。
我期望我开发的功能不用写文档。

## 一

以往我用 Rails 写 API 的话， 多数采用 grape + swagger 的方案，因为熟悉与简单。使用 grape + grape-entity 实现逻辑，用 grape-swagger + grape-swagger-rails 生成文档。

最近找了找用 Rails API Mode 而非 Grape 的方案下，维护 API 文档的可选项。写开发文档最容易挖的坑是没有跟随功能的变化及时进行更新。**过时的文档就像过时的地图，容易走到了死胡同才发现有问题**。要解决文档过时的问题，目前看来在 RSpec 里写文档最为简单合适，在写测试的同时完成文档的编写。

## 二

以下列举两个 **RSpec + Swagger** 方案。

### rswag

<https://github.com/rswag/rswag>
rswag 同时包含了 Swagger-based DSL 和 Swagger UI，是集成最方便的一个方案。README 里有较为详细的例子，支持从指定文件生成文档，支持生成多版本文档，如 API v1/v2 。如果是从零开始写文档，可以优先考虑。如果项目已经有较多 RSpec 测试，改动量会较大。

### rspec_api_documentation (RAD) + swagger_ui_engine

<https://github.com/zipmark/rspec_api_documentation>
<https://github.com/zuzannast/swagger_ui_engine>
此方案比 rswag 稍为复杂，因为文档信息生成与 Swagger UI 是分开配置的。swagger_ui_engine 负责把 Swagger UI 集成进 Rails。RAD 负责生成文档信息，支持 html_json_open_api 等十余种输出格式，其中 open_api 格式可用于 Swagger UI，json 格式可用于 Raddocs 或者 Apitome。如果只是需要简单的 API 文档，不用 swagger_ui_engine 也是可以的，RAD 可以直接生成简单的 HTML 以供查看。另外 RAD 支持定义不同的 group 从而生成不同版本的文档。

以上方案皆有诸多细节未一一列举，感兴趣的可以上 GitHub 自行搜索研究。

此外 Rails + Swagger 有一个 swagger-docs gem，在 controller 加 Swagger 代码，然而其已两年多未更新，只支持 Swagger 1.x 规格，过时已久，不需多看。

(END)
