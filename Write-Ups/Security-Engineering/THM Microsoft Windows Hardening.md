## Understanding General Concepts

#### Services

Windows Services create and manage critical functions such as network connectivity, storage, memory, sound, user credentials, and data backup and runs automatically in the background. These services are managed by the Service Control Manager panel and divided into three categories, i.e. Local, Network & System. Many applications like browsers and anti-virus software can also run their services for a seamless user experience.

#### Windows Registry

The Windows registry is a unified container database that stores configurational settings, essential keys and shared preferences for Windows and third-party applications. Usually, on the installation of most applications, it uses a registry editor for storing various states of the application. For example, suppose an application (malicious or normal) wants to execute itself during the computer boot-up process; In that case, it will store its entry in the Run & Run Once key.

Usually, a malicious program makes undesired changes in the registry editor and tries to abuse its program or service as part of system routine activities. It is always recommended to protect the registry editor by limiting its access to unauthorised users.

#### Event Viewer

Event Viewer is an app that shows log details about all events occurring on your computer, including driver updates, hardware failures, changes in the operating system, invalid authentication attempts and application crash logs. Event Viewer receives notifications from different services and applications running on the computer and stores them in a centralised database. 

Hackers and malicious actors access Event Viewer to increase their attack surface and enhance the target system's profiling. Event categories are as below:

- **Application:** Records events of already installed programs.
- **System:** Records events of system components.
- **Security:** Logs events related to security and authentication etc.

#### Telemetry

Telemetry is a data collection system used by Microsoft to enhance the user experience by preemptively identifying security and functional issues in software. An application seamlessly shares data (crash logs, application-specific) with Microsoft to improve the user experience for future releases.  

Telemetry functionality is achieved by Universal Telemetry Client (UTC) services available in Windows and runs through diagtrack.dll. Contents acquired through telemetry service are stored encrypted in a local folder %ProgramData%\Microsoft\Diagnosis and sent to Microsoft after 15 minutes or so.

### Q&A

What is the startup type of App Readiness service in the services panel?  

`Manual`

Open Registry Editor and find the key “tryhackme”. What is the default value of the key?   

`{THM_REG_FLAG}`

Open the Diagnosis folder and go through the various log files. Can you find the flag?  

`{THM_1000710}`

## Identity and Access Management


