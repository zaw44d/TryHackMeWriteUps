### What is Threat Modelling?

Threat modelling is a systematic approach to **identifying, prioritising, and addressing potential security threats** across the organisation. By simulating possible attack scenarios and assessing the existing vulnerabilities of the organisation's interconnected systems and applications, threat modelling enables organisations to develop proactive security measures and make informed decisions about resource allocation. 

Threat modelling aims to reduce an organisation's overall risk exposure by identifying vulnerabilities and potential attack vectors, allowing for adequate security controls and strategies. This process is essential for constructing a robust defence strategy against the ever-evolving cyber threat landscape.

### Threat, Vulnerability and Risk

As mentioned above, the main goal of threat modelling is to reduce the organisation's risk exposure. So before we deep dive into its application, let's review first the definitions of **Threat, Vulnerability and Risk**.

|**Type**|**Definition**|
|---|---|
|**Threat**|Refers to any potential occurrence, event, or actor that may exploit vulnerabilities to compromise information confidentiality, integrity, or availability. It may come in various forms, such as cyber attacks, human error, or natural disasters.|
|**Vulnerability**|A weakness or flaw in a system, application, or process that may be exploited by a threat to cause harm. It may arise from software bugs, misconfiguration, or design flaws.|
|**Risk**|The possibility of being compromised because of a threat taking advantage of a vulnerability. A way to think about how likely an attack might be successful and how much damage it could cause.|

To simplify it, we can use an analogy of an organisation as a house and describe the potential threat, vulnerability and risk.

|**Type**|**Analogy**|
|---|---|
|**Threat**|Occurrence of someone breaking inside your home and taking all your belongings.|
|**Vulnerability**|Weaknesses in your home security, such as broken locks or open windows.|
|**Risk**|Likelihood of being burglarised due to living in a neighbourhood with a high crime rate or a lack of an alarm system.|

Understanding the differences between threat, vulnerability, and risk is essential for effective threat modelling. It enables the organisation to effectively identify and prioritise security issues, resulting in a faster way of reducing risk exposure.

### High-Level Process of Threat Modelling

Before delving into different threat modelling frameworks, let's briefly run through a simplified, high-level process.

1. **Defining the scope**
    
    Identify the specific systems, applications, and networks in the threat modelling exercise.
    
2. **Asset Identification**
    
    Develop diagrams of the organisation's architecture and its dependencies. It is also essential to identify the importance of each asset based on the information it handles,  such as customer data, intellectual property, and financial information.
    
3. **Identify Threats**
    
    Identify potential threats that may impact the identified assets, such as cyber attacks, physical attacks, social engineering, and insider threats.
    
4. **Analyse Vulnerabilities and Prioritise Risks** 
    
    Analyse the vulnerabilities based on the potential impact of identified threats in conjunction with assessing the existing security controls. Given the list of vulnerabilities, risks should be prioritised based on their likelihood and impact.
    
5. **Develop and Implement Countermeasures**
    
    Design and implement security controls to address the identified risks, such as implementing access controls, applying system updates, and performing regular vulnerability assessments.
    
6. **Monitor and Evaluate**
    
    Continuously test and monitor the effectiveness of the implemented countermeasures and evaluate the success of the threat modelling exercise. An example of a simple measurement of success is tracking the identified risks that have been effectively mitigated or eliminated.
    

By following these steps, an organisation can conduct a comprehensive threat modelling exercise to identify and mitigate potential security risks and vulnerabilities in their systems and applications and develop a more effective security strategy. 

Remember that the example above is a generic high-level process; threat modelling frameworks will be introduced in the following tasks.

### Collaboration with Different Teams  

﻿The high-level process discussed above involves many tasks, so it is crucial to have multiple teams collaborate. Each unit offers valuable skills and expertise, helping improve the organisation's security posture. By collaborating, organisations can effectively address and align the security efforts needed to build a better defence.

In line with this, we will introduce the teams typically involved in a threat modelling Exercise.


|**Team**|**Role and Purpose**|
|---|---|
|**Security Team**|The overarching team of red and blue teams. This team typically lead the threat modelling process, providing expertise on threats, vulnerabilities, and risk mitigation strategies. They also ensure security measures are implemented, validated, and continuously monitored.|
|**Development Team**|The development team is responsible for building secure systems and applications. Their involvement ensures that security is always incorporated throughout the development lifecycle.|
|**IT and Operations Team**|IT and Operations teams manage the organisation's infrastructure, including networks, servers, and other critical systems. Their knowledge of network infrastructure, system configurations and application integrations is essential for effective threat modelling.|
|**Governance, Risk and Compliance Team**|The GRC team is responsible for organisation-wide compliance assessments based on industry regulations and internal policies. They collaborate with the security team to align threat modelling with the organisation's risk management objectives.|
|**Business Stakeholders**|The business stakeholders provide valuable input on the organisation's critical assets, business processes, and risk tolerance. Their involvement ensures that the efforts align with the organisation's strategic goals.|
|**End Users**|As direct users of a system or application, end users can provide unique insights and perspectives that other teams may not have, enabling the identification of vulnerabilities and risks specific to user interactions and behaviours.|

﻿Note that the list is not limited to these teams and may vary depending on your organisational structure. Moreover, the collaboration of these teams is not limited to threat modelling exercises, as they also work hand in hand in securing the organisation through different initiatives.

### Attack Trees

In addition to the high-level methodology discussed above, creating an attack tree is another good way to identify and map threats.

An ﻿attack tree is a graphical representation used in threat modelling to systematically describe and analyse potential threats against a system, application or infrastructure. It provides a structured, hierarchical approach to breaking down attack scenarios into smaller components. Each node in the tree represents a specific event or condition, with the root node representing the attacker's primary goal.

For a quick example, let's use the diagram below that represents a scenario of an attacker trying to gain unauthorised access to sensitive data stored in a cloud-based storage system.

In this diagram, the root node represents the attacker's primary goal: **gain unauthorised access to sensitive data**. The first level of child nodes represents different high-level strategies an attacker might do to achieve the goal. Each node further breaks down into specific steps, detailing the attacker's possible techniques and actions.

In addition to the traditional hierarchical structure, attack trees can be organised as attack paths, which depict the possible routes or sequences of vulnerabilities a threat actor can exploit to achieve their goal. Attack paths are essentially chains of vulnerabilities that are interconnected.  

In an attack path representation, the initial starting node represents the attacker's entry point into the system or network. From there, the various branches or nodes represent the specific vulnerabilities, attack vectors, or steps the threat actor can follow to advance towards their objective.

### Q&A

What is a weakness or flaw in a system, application, or process that can be exploited by a threat?

`vulnerability`

Based on the provided high-level methodology, what is the process of developing diagrams to visualise the organisation's architecture and dependencies?

`Asset Identification`

What diagram describes and analyses potential threats against a system or application?

`Attack Tree`

## Modelling with MITRE ATT&CK
