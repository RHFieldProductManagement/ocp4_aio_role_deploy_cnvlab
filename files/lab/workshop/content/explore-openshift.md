#### Command Line Interface (CLI)

OpenShift includes a feature-rich web console with both an *Administrator* and a *Developer* perspective. In addition to the web console, OpenShift includes command line tools to provide users with a comprehensive interface to work with applications deployed on the platform.  The `oc` command line tool is an executable built on-top of `kubectl` and is written in the Go programming language. It is available for Windows, MacOS, and Linux.

This lab environment has the `oc` command line tool installed, and your lab user is already logged in to the OpenShift cluster with administrative access - you're not sharing this cluster with anyone else. As an example, issue the following command to see help information for the CLI:

```execute-1
oc help
```

#### Using a Project

Projects are a top level concept to help you organise your resources. An OpenShift project allows a community of users (or a single user) to organise and manage their content/resources in isolation from other communities. Each project has its own resources, policies (who can/cannot perform actions), and constraints (quotas and limits on resources, etc). Projects act as a "wrapper" around all the application services and endpoints you, or your teams, are using for your work.

#### Web Console

OpenShift ships with a web-based console that will allow users to perform various tasks via a web browser. From within the lab guide window you'll see two buttons at the top of the screen, with the currently active tab in blue, one for the "**Terminal**", the default, and one for the "**Console**"; you can switch between these two options at will. Select the console and you should see the OpenShift web console, in which you'll already be logged in:

<img  border="1" src="img/console-button-new.png"/>

With the Web Console provided in this lab environment, we will be able to execute the majority of the commands and we should be able to follow most of the lab sections. However, once we install the OpenShift Virtualization operator, this will add some "extra" graphical interfaces in the console that, unfortunately, cannot be displayed. But no need to worry, all we need to do is open a **dedicated** tab/window for the real full OpenShift Web Console.

To do so, simply open a new window/tab in the browser you are using for this lab and navigate to this link: %cnvlab-console-url%. To login to the console, you will need to type "**kubeadmin**" (the OpenShift administrator user) as the user and paste its password: %cnvlab-kubeadmin-password%. 

The first time you access the web console, you will most likely be in the *Administrator* perspective. At the top of the left navigation menu, you can toggle between the Administrator perspective and the Developer perspective.

<img  border="1" src="img/explore-dev-view-new.png"/>

Select **Developer** to switch to the Developer perspective. Once the Developer perspective loads, you should select the **Topology** view from the menu on the left-hand side. Right now, there are no applications so all you will see is a list of OpenShift projects already created. Once you begin working on the lab, you'll be able to visualise and interact with the components in your application here.

We will be using a mix of command line tooling and the web console for the labs. Select "**Validate**" below to continue with the validation of the environment and to become more familiar with OpenShift and its core functionality.
