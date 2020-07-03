Checklist for Server Account Creation:

- [ ] Make sure that the user requesting for a server account has Prof's explicit approval. One doesn't get a server account without his approval.
- [ ] Share this [form](https://forms.office.com/Pages/ResponsePage.aspx?id=vDsaA3zPK06W7IZ1VVQKHGN1C-0xgJ9OtHFVyUim7tBUNDJDQ1ZDSlNRTjVHT1JMRVRWVllCVE00VS4u) with the user to collect his/her details, ideally by email and `rrc-sysads@lists.iiit.ac.in` and Prof. should be CC'ed.
- [ ] Create the account by logging into `ipa.rrc.iiit.ac.in` and firing the [create-account.sh](https://github.com/RoboticsIIITH/sysad-scripts/blob/master/ipa/create-account.sh) script. Example usage: `kinit karnik && ./create-account.sh user@email username 912346789`. The script should mostly run without problems as long as the user's email address (intern or student) is registered with the IIIT LDAP. But if there is a problem, run the lines manually. 
- [ ] Assign the user to a particular Users group on web IPA (neon, green, blue) according to current user distribution and activity. Without this the user won't be able to login into the server. The servers are almost identical in terms of software stack, so the user's preference shouldn't matter unless he or she wants to share files on the `/scratch` space with someone else.
- [ ] Inform the user about the assigned Users group. The user would've already received the account credentials from [create-account.sh](https://github.com/RoboticsIIITH/sysad-scripts/blob/master/ipa/create-account.sh).
- [ ] Invite the user to this Github organization, under the `Global Read` team for access to the Wiki and sysad-scripts repositories.
- [ ] Make a log about this on #server-log on Discord.
