# Deletion

Depositors can request deletion of individual files or entire intellectual objects through the Pharos Web UI, but not through the API. This is by design, as we want to prevent large-scale accidental automated deletion.

When a depositor requests a deletion, Pharos sends an email to all admin users at the depositing institution saying that the user has requested a deletion and asking the admin to approve. The email names the user who requested the deletion and lists the files or objects the user wants to delete. It also includes a link the admin can click to approve the deletion.

No deletions will proceed without approval from the institutional admin. Some smaller institutions may have only one active administrator. In those cases, the person who approves the deletion may be the same person who requested it. APTrust considers it the depositor's duty to ensure they trust this person to manage deletions unchecked, or to add a second approver if they don't.

When an admin clicks the link to approve a deletion, Pharos ensures the admin either has an active, valid, authenticated session, or it forces them to log in. This prevents non-authorized users from using the link to approve deletions.

The deletion process follows the steps outlined under bulk deletion below, except 1) it is initiated directly by an institutional user, and 2) it does not require the approval of an APTrust administrator.

## Bulk Deletion

Pharos includes an endpoint to initiate bulk deletions. This endpoint is available only to APTrust administrators. The bulk deletion process follows these steps:

1. The depositor sends APTrust a list (usually a spreadsheet) of files and/or objects they want to delete.
2. An APTrust administrator feeds the list to the bulk deletion initiation endpoint in Pharos.
3. Pharos emails all administrators at the depositing institution saying that the initiating user has requested a bulk deletion. The email includes an attachment listing the items to be deleted.
4. The institutional admin clicks a link to approve the bulk deletion.
5. Pharos emails APTrust admins, telling them the institution has approved the bulk deletion.
6. The institutional admin approves the bulk deletion.
7. Pharos creates a deletion WorkItem for each file to be deleted.
8. A cron job that runs every 10 minutes or checks for new deletion requests and copies the WorkItem IDs of those requests to `apt_file_delete` topic in NSQ.
9. The `apt_delete` worker checks that all required approvals have been granted, and if so, deletes the item.
10. `apt_delete` creates a PREMIS event in Pharos describing when the file was deleted, who requested the deletion, and who approved it.
11. `apt_delete` marks the WorkItem and NSQ message complete.
