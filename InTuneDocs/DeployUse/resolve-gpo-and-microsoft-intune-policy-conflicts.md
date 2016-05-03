---
# required metadata

title: Resolve GPO and Intune policy conflicts | Microsoft Intune
description:
keywords:
author: robstackmsft
manager: jeffgilb
ms.date: 04/28/2016
ms.topic: article
ms.prod:
ms.service: microsoft-intune
ms.technology:
ms.assetid: e76af5b7-e933-442c-a9d3-3b42c5f5868b

# optional metadata

#ROBOTS:
#audience:
#ms.devlang:
ms.reviewer: jeffgilb
ms.suite: ems
#ms.tgt_pltfrm:
#ms.custom:

---

# Resolve Group Policy Objects (GPO) and Microsoft Intune policy conflicts
Intune uses policies that help you manage settings on the computers you manage. For example, you could use a policy to control settings for the Windows Firewall on computers. Many of the Intune settings are similar to settings you might configure with Windows Group Policy. However, it is possible that, at times, the two methods might conflict with one another.

When conflicts happen, domain-level Group Policy takes precedence over Intune policy, unless the computer can’t log on to the domain. In this case, Intune policy is applied to the client computer.

## What to do if you are using Group Policy
Check that any policies you apply are not being managed by Group Policy. To help prevent conflicts, you could employ one or more of the following methods:

-   Move your computers to an Active Directory organizational unit (OU) that does not have Group Policy settings applied before you install the Intune client. You can also block Group Policy inheritance on OUs that contain computers enrolled in Intune to which you do not want to apply Group Policy settings.

-   Use a WMI filter, or security filter to restrict GPOs only to computers that are not managed by Intune. For information and examples of how to do this, see the [How to filter existing Group Policy Objects to avoid Conflicts with Microsoft Intune policy](resolve-gpo-and-microsoft-intune-policy-conflicts.md#BKMK_Filter) section below.

-   Disable or remove the Group Policy Objects that conflict with the Intune policies.

For more information about Active Directory and Windows Group Policy, see your Windows Server Documentation.

## How to filter existing GPOs to avoid conflicts with Intune policy
If you have identified GPOs with settings that conflict with Intune policies, you can use either of the following filtering methods to restrict those GPOs only to computers that are not managed by Intune.

### Use WMI filters
WMI filters selectively apply GPOs to computers that satisfy the conditions of a query. To apply a WMI filter, deploy a WMI class instance to all computers in the enterprise before you enroll any computers in the Intune service.

#### To apply WMI filters to a GPO

1.  Create a management object file by copying and pasting the following into a text file, and then saving it to a convenient location as **WIT.mof**. The file contains the WMI class instance that you deploy to computers that you want to enroll in the Intune service.

    ```
    //Beginning of MOF file.
    #pragma classflags("forceupdate")
    #pragma namespace ("\\\\.\\Root")
    instance of __Namespace
    {
       Name = "WindowsIntune";
    };

    #pragma namespace ("\\\\.\\Root\\WindowsIntune")
    [
       Description("This class defines Microsoft Intune common properties")
    ]
    class WindowsIntune_ManagedNode
    {
       [ read, Description("This defines whether Microsoft Intune Policy is enabled"): DisableOverride ToSubClass ]
       boolean WindowsIntunePolicyEnabled;
       [ read, key, Description("This property defines the version." "Example: 1.0"): ToSubClass ]
       string Version;
    };

    instance of WindowsIntune_ManagedNode
    {
       Version = "1.0";
       WindowsIntunePolicyEnabled = 1;
    };
    ```

2.  Use either a startup script or Group Policy to deploy the file. The following is the deployment command for the startup script. The WMI class instance must be deployed before you enroll client computers in the Intune service.

    **C:/Windows/System32/Wbem/MOFCOMP &lt;path to MOF file&gt;\wit.mof**

3.  Run either of the following commands to create the WMI filters, depending on whether the GPO you want to filter applies to PCs that are managed by using Intune or to computers that are not managed by using Intune.

    -   For GPOs that apply to computers that are not managed by using Intune, use the following:

        ```
        Namespace:root\WindowsIntune
        Query:  SELECT WindowsIntunePolicyEnabled FROM WindowsIntune_ManagedNode WHERE WindowsIntunePolicyEnabled=0
        ```

    -   For GPOs that apply to computers that are managed by Intune, use the following:

        ```
        Namespace:root\WindowsIntune
        Query:  SELECT WindowsIntunePolicyEnabled FROM WindowsIntune_ManagedNode WHERE WindowsIntunePolicyEnabled=1
        ```

4.  Edit the GPO in the Group Policy Management console to apply the WMI filter that you created in the previous step.

    -   For GPOs that should apply only to computers that you want to manage by using Intune, apply the filter **WindowsIntunePolicyEnabled=1**.

    -   For GPOs that should apply only to computers that you do not want to manage by using Intune, apply the filter **WindowsIntunePolicyEnabled=0**.

For more information about how to apply WMI filters in Group Policy, see the blog post [Security Filtering, WMI Filtering, and Item-level Targeting in Group Policy Preferences](http://go.microsoft.com/fwlink/?LinkId=177883).

### Use security group filters
Group Policy lets you apply GPOs to only those security groups specified in the **Security Filtering** area of the Group Policy Management console for a selected GPO. By default, GPOs apply to **Authenticated Users**.

-   In the **Active Directory Users and Computers** snap-in, create a new security group that contains computers and user accounts that you do not want to manage by using Intune. For example, you could name the group **Not In Microsoft Intune**.

-   In the Group Policy Management console, on the **Delegation** tab for the selected GPO, right-click the new security group to delegate appropriate **Read** and **Apply Group Policy** permissions to both users and computers in the security group. (**Apply Group Policy** permissions are available on the **Advanced** dialog box.)

-   Then apply the new security group filter to a selected GPO, and remove the **Authenticated Users** default filter.

The new security group must be maintained as enrollment in the Intune service changes.

### See also
[Manage Windows PCs with Microsoft Intune](manage-windows-pcs-with-microsoft-intune.md)