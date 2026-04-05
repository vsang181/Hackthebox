## Engagement Closeout

When the testing period ends, send the client a notification email immediately. The email should confirm the end of testing and give a specific timeframe for report delivery. You can also use this communication to ask for availability for a report review meeting, though this is often better handled when delivering the report itself. If you have communicated clearly throughout the engagement, the client should already have a broad picture of what to expect. Never let a client read about the highest-risk findings for the first time in the report.

## Attack Path Recap

Write out the full attack path from initial access to domain compromise before closing out. This serves two purposes: it helps you visualise the chain of exploitation and identify any findings you may have missed, and it provides a clean narrative for the report. The recap should show the path of least resistance only. Do not include every failed attempt, detour, or tool tried along the way, as this clutters the chain and makes it difficult for the reader to follow.

Some firms structure their reports as a full narrative walkthrough, calling out specific findings at each step. The format differs between companies, but having this list available is always useful, particularly if the client later asks how a specific access was achieved.

## Structuring Findings

Ideally, findings have been noted down continuously throughout the engagement with command output and screenshots captured in a notetaking tool. Before access to the internal network ends, confirm that:

- All findings are recorded with supporting evidence
- Findings are ordered from highest to lowest risk
- No critical evidence or command output is missing
- No additional scans or access are needed to fill gaps

Do not rely on being able to ask the client for re-access to gather evidence that should have been collected during testing. Refer to the [Documentation and Reporting module](https://academy.hackthebox.com/module/162/section/1533) for detailed guidance on setting up a notetaking environment and structuring evidence throughout the engagement. That module also includes a sample report that can be used as a template for writing up the findings from this lab.

## Post-Engagement Cleanup

Throughout the engagement, keep a running log of every action that may require cleanup:

- Scans and attack attempts performed
- Files uploaded to target systems (tools, shells, payloads, notes)
- Accounts created or compromised
- Configuration changes made

Before the engagement formally closes, remove all uploaded files and restore any changes to the state they were in before testing began. Even if full cleanup is completed, every change and file upload must still be documented in the report appendices. Retain all logs and a detailed activity timeline for a period after the assessment ends so that the client can correlate any security alerts they received against your testing activity.

## Client Communication

Notify the client the moment testing ends so they can attribute any residual alerts correctly. During the reporting phase, maintain clear communication by providing a firm report delivery date and a brief summary of findings if requested. Expect to attend a report review meeting to walk the client through the results. Key points to manage during this phase:

- If retesting is included in the Scope of Work, agree on a remediation timeline with the client
- The client may not have a remediation timeline ready immediately and may reach out later regarding post-remediation testing
- The client may contact you over the following days or weeks to ask about specific alerts, so keep notes and logs accessible

## Internal Project Closeout

Once the report is delivered and the closeout meeting is complete, the following internal tasks are typically performed:

- Archive the report and all associated project data on a company share drive
- Hold a lessons learned debrief to identify what went well and what can be improved
- Complete any post-engagement questionnaires required by the sales team
- Handle administrative tasks including invoicing

If post-remediation testing cannot be performed by the original tester due to scheduling, an internal knowledge transfer to a teammate is required before closing the project fully.

## Next Steps

After completing the module, return to the lab without the walkthrough and work through it independently. Build a list of all skills and associated tools covered and revisit any areas that still present difficulty. Use the lab to:

- Try different tools and techniques to achieve the same objectives
- Practice documentation and note-taking throughout a full simulated engagement
- Produce a complete commercial-grade penetration testing report using the sample from the [Documentation and Reporting module](https://academy.hackthebox.com/module/162/section/1533)
- Practice oral presentation skills by walking a colleague or friend through your findings as if it were a real debrief

Consider working through one or more [Hack The Box Pro Labs](https://www.hackthebox.com/hacker/pro-labs), approaching each as a full simulated penetration test to build engagement skills as much as technical ones.
