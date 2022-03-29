Application Live View shows an individual running process, for example, a Spring Boot application deployed as a workload resulting in a JVM process running inside of a Pod. This is an important concept of Application Live View: **only running processes are recognized** by Application Live View. If there is not a running process inside of a running Pod, Application Live View does not show anything.

Under the hood, Application Live View uses the concept of **Spring Boot Actuators** to gather data from those running processes. 
Application Live View **does not store any of that data** for further analysis or historical views. 

The Application Live View is available for a desired Pod from the Pods section under Runtime Resources tab. In addition it's currently also still available via on the /app-live-view subpage of the gui

The ootb supply chains will set the required labels to enable Application Live View automatically.
```terminal:execute
command: kubectl get kservice spring-sensors -n dev-space -o yaml
clear: true
```