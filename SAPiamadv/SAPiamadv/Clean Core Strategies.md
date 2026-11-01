![[MCC100_U2L2_001.png]]
### Introduction

Embarking on the journey of adopting SAP Cloud ERP requires an important choice for customers. This choice is whether to opt for a new implementation ("greenfield") or a system conversion ("brownfield") implementation strategy. A third option (selective data transition | system landscape conversion) also exists. Let's talk about these options in the context of clean core principles.

### Understanding the Different Types of Implementations

|     |                                                                                                                                                                                |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
|     | SAP Cloud ERP is a ready-to-run solution. Both SAP Cloud ERP Private and on-Premise ERP can be considered as tailored-to-fit solutions for enterprises who need that approach. |

To start, we must grasp the three deployment models that are available to customers when considering SAP ERP. Two of the options are cloud deployments and the third is an on-premise deployment option:

- SAP S/4HANA Cloud public edition
- SAP S/4HANA Cloud private edition
- On-premise ERP (when SAP S/4HANA is deployed on-premise)

The term "SAP Cloud ERP" can refer to the public, private (or both) cloud deployment options depending on the context. The term "SAP Cloud ERP Private" can be used when needing to refer to the specific private cloud deployment option. In addition when we refer to SAP Cloud ERP Private assume the information applies equally to on-premise ERP. If a distinction is necessary it will be explicitly stated.

Next, we must grasp the concepts of different types of implementations. A greenfield implementation refers to a new installation. This new implementation can be any of the aforementioned deployment options. On the other hand, a brownfield implementation involves upgrading an existing system to SAP Cloud ERP Private. You then implement a detailed process to determine which custom code is retained, and adjust and adapt that custom code. It is important to note that the public deployment option cannot be chosen in the context of a brownfield approach.

Imagine that we are developers and a medium-sized manufacturing company hires us, eager to adopt SAP ERP. The essential principles that guide the company's eagerness are simplicity, flexibility, and a modular design that can easily change with business demands. These principles comprise the "lighthouse" for our "clean core" strategy.

A greenfield approach means that we start on a clean slate, designing and building in a newly installed or provisioned system (based on any of the three deployment options ). It's like constructing a new building on a fresh piece of land. There are no limitations from previous architecture, and we have full liberty to design our processes based on current and future business needs.

Now, consider a different scenario. As developers, we're asked to modernize our organization's existing inventory management system (based on on-Premise ERP) by migrating to SAP Cloud ERP Private. At the same time we will preserve particular business processes that are central to our competitive advantage. Here, the brownfield approach excels. Just like renovating an existing building, a brownfield implementation involves us converting our current on-Premise ERP system to SAP Cloud ERP Private. This preserves the original system's data and configurations.

### The Impact of Greenfield or Brownfield on Clean Core Strategies

So, how does the choice between greenfield or brownfield affect companies clean core strategies? In the previous lesson, we saw that clean core refers to an up-to-date system on the latest release. The system includes cloud-compliant extensions and integration, with optimal master data quality and with perfect process design. It ultimately simplifies system architecture, increasing the efficiency and speed of operations.

In the first manufacturing company scenario, let's say the existing system has been in place for years and is packed with custom code, redundant functions, and outdated modules. Adopting the greenfield approach allows us to start fresh, adopting a clean core from the beginning. This approach ensures efficiency and reduces complexity. As we've mentioned, a new system starts by default with a clean core.

On the contrary, a brownfield approach, which is our preferred method in the second inventory management scenario, allows the target SAP Cloud ERP Private instance to retain some of the previous system's complexities. This approach may present a challenge. Since we're retaining the existing system we will inherit historic complexities and possibly unwanted customizations. However, in spite of that challenge, SAP Cloud ERP Private can still be "clean" in the case of system conversions. To get there we engage in a deliberate process to systematically make it clean. To sum up, the clean core strategy in that case lies in us identifying which aspects of the legacy system to retain, eliminate, or transform and then to do so. This requires us to have a strong understanding of our current system's functions and a clear idea of how adopting SAP Cloud ERP Private can achieve the desired changes. It challenges us to reconcile the old with the new-efficiently transforming and adapting everything (including but not limited to custom code) so that it is clean core compliant.

Two important things to note. First is that although not explicitly discussed a selective data transition | system landscape transformation is a third option. This approach allows customers to consolidate multiple ERP systems (due to a history of mergers and acquisitions for example) into one SAP Cloud ERP Private target. Second while the hypothetical discussed clean core and implementation strategies from a developer point of view it should not be assumed that developers are the single guardian of clean core. They are not. Enterprise architects have a central role in promoting and guaranteeing a clean core strategy is defined and adopted by organizations.

|     |                                                                                                                                                        |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
|     | Clean core can be accomplished with all implementation approaches. The approach chosen by the customer only affects the specifics of how to get there. |

### Business Processes Are an Important Factor to Consider

There are several factors to consider when deciding whether to pursue a greenfield or brownfield approach. We examine one factor, business process considerations, here. For a complete discussion of all the various factors to consider in choosing which approach to take, see the lesson ["Evaluating Between a System Conversion or a New Implementation"](https://learning.sap.com/courses/practicing-clean-core-extensibility-for-sap-s-4hana-cloud/evaluating-between-a-system-conversion-or-a-new-implementation_d8ae2a25-1a52-43fc-8977-51b2ac714026) in the learning journey ["Practicing Clean Core Extensibility For SAP S/4HANA Cloud"](https://learning.sap.com/learning-journeys/practicing-clean-core-extensibility-for-sap-s-4hana-cloud).

Continuing with our first (greenfield) scenario, if the manufacturing company is willing and able to adopt new business processes based on current best practices with a fit to standard approach a greenfield approach becomes viable. With this strategy, they have the freedom to initiate a clean core strategy without the restrictions of legacy coding and design.

Alternatively, continuing with our second (brownfield) scenario, if the inventory management system uses complex business processes that are necessary and are expensive or challenging to redesign from the beginning, a brownfield approach becomes viable. A clean core as always is possible and the goal. In this case, critical business processes are preserved.

### Balancing Business Requirements and Clean Core Strategies

How we apply clean core principles in migrating to SAP Cloud ERP ultimately depends on a balance between business requirements and implementation strategies. Our role as developers extends beyond writing code. We become critical intermediaries in ensuring the upgraded system both aligns with business requirements and adheres to clean core principles. A greenfield strategy presents the simplest and most direct route to achieving a clean core, but is not always appropriate. A brownfield strategy, while it can preserve some complexity due to highly specific business requirements, still offers a medium to transform and simplify existing codes in a clean code-compliant way.

Ultimately, the choice between greenfield and brownfield correlates closely with the business context, availability of resources, and the tolerance for comprehensive change. As developers, understanding these strategies' impacts on clean core principles equips us in making informed decisions for the business.

### Conclusion

Both Greenfield and Brownfield implementations have their potential in the right setup. Understanding these options and the part they play in maintaining a clean core can significantly influence the success of your SAP Cloud ERP project. Remember, whichever approach you choose, the goal is to promote a clean core strategy. A clean core strategy calls for an optimized, simplified, and flexible system that effectively supports your organization's unique business processes while facilitating innovation and agility.

|     |                                                                                                                                                                                  |
| --- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
|     | With a clean core strategy, you can help ensure your organization benefits from the latest release with minimal modifications and cloud-compliant extensions and customizations. |