In our current system, all users have unrestricted access to all resources within the application. This lack of role-based controls means that there is no differentiation in access levels, regardless of a user’s role or responsibilities.

**Introduction**

-   **Overview**:

    -   Role-based access control in a multi-group environment.

    -   Approval workflows at system and group levels.

    -   Handling restricted datasets and ensuring data security.

-   **Key Points**:

    -   Defined roles for **System Administrators (SysAdmin)** and **Group Owners (GAdmin)**.

    -   Request and approval workflows.

    -   Notifications, logging, and timeline management.

**Defined Roles in the System**

-   **System Administrator (SysAdmin)**:

    -   Supreme user managing system-wide settings, approvals, and compliance.

    -   Can override group-level actions and review all system logs.

-   **Group Owner (GAdmin)**:

    -   Responsible for managing datasets and users within their group.

    -   Handles group-specific requests and approvals.

-   **Group Members**:

    -   Users within a group assigned roles like **Read-Only**, **Write**, or **Admin**, controlling their access to datasets.

**Initial Access and Permission Requests**

-   **Default Access**: New users receive read-only access.

-   **Request Process**:

    -   Users request write or admin permissions via the system.

    -   **SysAdmins** manage system-wide requests; **GAdmins** manage group-specific requests.

-   **Approval and Role Assignment**: After review, users are assigned their role, allowing them to create and access datasets.

**Dataset Creation and Group Access**

-   **Dataset Assignment**: Datasets created by users are automatically assigned to the user's group.

-   **Group-Level Access**:

    -   **Read-Only**: View-only access.

    -   **Write**: Edit datasets.

    -   **Admin**: Full group control.

-   **System-Level Oversight**: **SysAdmins** monitor dataset access and usage across all groups.

**Inter-Group Access Requests**

-   **Scenario**: Group 1 member requests access to Group 2 datasets.

-   **Request Handling**:

    -   The request is routed to **Group 2’s GAdmin** for approval.

    -   **Notification**: GAdmin receives an email or in-app notification.

-   **Approval Options**:

    -   **Approve**: Group 1 users gain access to Group 2 datasets. \*\*\*

    -   **Reject**: The request is denied.

    -   **Request Changes**: GAdmin requests clarification or modifications.

**Handling Restricted Datasets**

-   **Restricted Datasets**: Sensitive datasets requiring extra approval for changes, even from users with write access.

-   **Approval Process**:

    -   **GAdmin** reviews changes made to restricted datasets.

    -   **Approval or Rejection**: The changes are published if approved.

    -   **Additional Information**: GAdmin can request clarification if needed.

-   **SysAdmin Involvement**: If needed, **SysAdmins** can also review changes to system-critical datasets.

**Managing Requests and Notifications**

-   **Request Management**:

    -   Requests are tracked and assigned a **unique ID** for auditing and follow-up.

    -   **Timeline for Action**: Requests are set to be approved or rejected within a defined timeframe.

    -   **Escalation**: If not addressed, requests are escalated to a higher-level admin.

-   **Notifications**:

    -   **GAdmins** and **SysAdmins** are notified via email or in-app for critical actions (e.g., restricted dataset changes).

    -   **User Notification**: Users are informed of the request's status (approved, rejected, or modified).

**Logging, Monitoring, and Compliance**

-   **Comprehensive Logging**:

    -   Every action, request, and approval is logged.

    -   **SysAdmins** and **GAdmins** can view detailed logs of dataset access and modifications.

-   **Security and Compliance**:

    -   **Activity Timeline**: All actions are timestamped, showing a clear trail of who accessed or changed what.

**Notifications and Timeline Management**

-   **Multi-Channel Notifications**:

    -   Admins receive real-time notifications via email, SMS, or in-app for urgent actions.

-   **Request Tracking and Timeline**:

    -   Each request is tracked, and admins can view the entire **activity timeline**.

    -   **Escalation**: Requests exceeding the time limit are escalated to a higher authority.

-   **SLA-Based Approval**: For urgent requests, such as critical dataset changes, SLA-based timelines ensure quick action.

**Scalability and Future Improvements**

-   **Large Dataset Management**:

    -   Use **batch approval** processes for faster handling of large datasets or frequent requests.

-   **Self-Service Dashboard**:

    -   Users can view the status of their requests, while **SysAdmins** and **GAdmins** can track pending approvals.

**Summary**

-   **Clear Role Definitions**: **SysAdmins** manage system-wide requests and compliance, while **GAdmins** handle group-specific activities and approvals.

-   **Seamless Workflow**: Data access, request handling, and restricted dataset changes are all managed through a structured, role-based approval process.

-   **Security and Transparency**: Detailed logging, real-time notifications, and escalation processes ensure the system remains secure and compliant.

