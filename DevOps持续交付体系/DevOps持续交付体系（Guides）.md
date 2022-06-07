DevOps持续交付体系（Guides）
============

## 文化篇
- 软件工程的发展
    - 瀑布式开发
    - 精益运动
    - 敏捷开发
    - 持续交付运动
    - DevOps持续交付体系
  
  [该章节文档链接…](https://github.com/yaocoder/MyDocument/blob/master/DevOps%E6%8C%81%E7%BB%AD%E4%BA%A4%E4%BB%98%E4%BD%93%E7%B3%BB/%E6%96%87%E5%8C%96/%E8%BD%AF%E4%BB%B6%E5%B7%A5%E7%A8%8B%E7%9A%84%E5%8F%91%E5%B1%95.md)

- DevOps文化
    - DevOps的组织文化
      - 交付“价值”
      - 允许失败、获得反馈、持续改善
      - 持续学习、彼此信任
    - DevOps的研发文化
      - 目标：高质量、低风险地快速发布软件价值
      - 核心模式
      - 核心原则

    [该章节文档链接…](https://github.com/yaocoder/MyDocument/blob/master/DevOps%E6%8C%81%E7%BB%AD%E4%BA%A4%E4%BB%98%E4%BD%93%E7%B3%BB/%E6%96%87%E5%8C%96/DevOps%E6%96%87%E5%8C%96.md)
    
## 流程及指导篇
### IT价值流
[IT价值流概览链接…](https://github.com/yaocoder/Architect-CTO-growth/blob/master/DevOps%E6%8C%81%E7%BB%AD%E4%BA%A4%E4%BB%98%E4%BD%93%E7%B3%BB/%E6%B5%81%E7%A8%8B%E5%8F%8A%E6%8C%87%E5%AF%BC/IT%E4%BB%B7%E5%80%BC%E6%B5%81%E6%A6%82%E8%A7%88.md)
- 一、产品准备期
    - 0.业务需求协作管理
    - 1.需求拆分（产品组织主导）
      - 1.1 需求拆分的好处
      - 1.2 需求拆分的方法        
        - 1.2.1 以用户故事进行需求拆分
        - 1.2.2 用户故事的拆分原则（INVEST）
    - 2.研发准备工作（研发组织）
      - 2.1 研发任务拆分
      - 2.2 测试用例编写
      - 2.3 系统架构梳理
        - 2.3.1 架构的设计原则
        - 2.3.2 涉及遗留架构的改造策略

  [该章节文档链接…](https://github.com/yaocoder/Architect-CTO-growth/blob/master/DevOps%E6%8C%81%E7%BB%AD%E4%BA%A4%E4%BB%98%E4%BD%93%E7%B3%BB/%E6%B5%81%E7%A8%8B%E5%8F%8A%E6%8C%87%E5%AF%BC/%E4%BA%A7%E5%93%81%E5%87%86%E5%A4%87%E6%9C%9F.md)
  
- 二、产品交付期
    - 1.持续集成（Continuous integration  简称CI）
      - 1.1 持续集成的步骤
      - 1.2 持续集成的实践]
        - 1.2.1 使构建过程脚本化，搭建持续集成框架
        - 1.2.2 向构建中添加已有的自动化验证集合
        - 1.2.3 选择利于持续集成的分支策略
        - 1.2.4 建立六步提交法
    - 2.持续测试
      - 2.1 Brian Marick 测试四象限
      - 2.2 测试类型
        - 2.2.1 单元测试
        - 2.2.2 组件测试
        - 2.2.3 集成测试
        - 2.2.4 验收测试
    - 3.持续部署（发布）
      - 3.1 高频发布的好处
      - 3.2 低风险发布策略
        - 3.2.1 蓝绿部署
        - 3.2.2 滚动部署
        - 3.2.3 灰度发
    - 4.软件配置管理
      - 4.1 核心原则
      - 4.2 制品库管理
        - 4.2.1 临时软件包库
        - 4.2.2 正式软件包库
        - 4.2.3 外部软件包库
      - 4.3 数据版本管理
    - 总：部署流水线
      - 1.核心阶段
        - 基础环境准备
        - 1.1 “提交构建”阶段
        - 1.2 “UAT验收测试”阶段
        - 1.3 “部署/发布”阶段
      - 2.基础支撑服务
      - 3.需遵守的原则

  [该章节文档链接…](https://github.com/yaocoder/Architect-CTO-growth/blob/master/DevOps%E6%8C%81%E7%BB%AD%E4%BA%A4%E4%BB%98%E4%BD%93%E7%B3%BB/%E6%B5%81%E7%A8%8B%E5%8F%8A%E6%8C%87%E5%AF%BC/%E4%BA%A7%E5%93%81%E4%BA%A4%E4%BB%98%E6%9C%9F.md)
  
- 三、产品运营期
    - 1.监测与决策](#1监测与决策)
      - 1.1 生产环境监测体系
        - 1.1.1 后台服务监测
        - 1.1.2 分发客户端监测
      - 1.3 问题处理体系
    - 2.数据治理体系

  [该章节文档链接…](https://github.com/yaocoder/Architect-CTO-growth/blob/master/DevOps%E6%8C%81%E7%BB%AD%E4%BA%A4%E4%BB%98%E4%BD%93%E7%B3%BB/%E6%B5%81%E7%A8%8B%E5%8F%8A%E6%8C%87%E5%AF%BC/%E4%BA%A7%E5%93%81%E8%BF%90%E8%90%A5%E6%9C%9F.md)
