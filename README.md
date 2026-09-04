
Requirements Gathering

We don't currently have a separate environment to test golden image or hardening changes before they reach the developer environment. This project creates a pre-prod environment where those changes can be validated first.

The environment needs to support:

- STIG compliance testing: Use Nessus compliance scans with the DISA STIG audit files, the same way we plan to scan production systems.
- Credentialed vulnerability scanning: Each VM will need a working scan service account. That work is being handled in a separate project and should be linked here as a dependency.
- Golden image testing: Validate both image types separately:
  - FIPS-enabled image
  - DEFAULT crypto policy image used for the Jira and Confluence compatibility exception
- Basic functional testing: Confirm services start normally, TLS works, domain join/authentication works, and GitLab Runner can register successfully.

We are not expecting a separate Azure subscription, so the environment will be logically separated within the existing subscription. The resource group, VNet, subnet, NSGs, and VMs are already in place.

Plan Development

Start by reviewing the separation between the test and developer environments. This includes subscription-level RBAC, identities, deployment accounts, Key Vault, IaC state, and DNS.

Next, separate the monitoring and logging. The test environment should have its own Log Analytics workspace and Data Collection Rules so test activity does not mix with the developer environment.

Once the scan service accounts are available, add the test VMs to Nessus and run credentialed baseline scans against both image types. These results will be used as the baseline for future image changes.

After that, set up the image promotion process:

New image version → deploy to test → run Nessus scans and smoke tests → promote for use in the developer environment.

New versions should not be marked as "latest" until testing is complete.

Open item: Confirm whether the additional test VMs require more Nessus scanner capacity or licensing. If not, no purchase request is needed.

Cross-Project Blocker

The scan service account ticket should be linked to this issue as:

is blocked by → [service account issue key]

It would also be helpful to put the issue key near the top of the description so the dependency is easy to see.

The Jira issue link itself does not normally prevent someone from moving the ticket forward. Unless we already have something like ScriptRunner or JSU configured to enforce it, the simplest option is to treat completion of the service account ticket as an exit requirement for Cybersecurity Review.

That keeps the dependency visible without adding additional Jira workflow changes.




-‐-----—--------------
1. **SSH daemon config**: Use an `sshd_config.d` drop-in instead of replacing the main config. Replacing it can remove GitLab's `AuthorizedKeysCommand` and break Git over SSH.

2. **SSH access restrictions**: If we use `AllowGroups` or `DenyUsers`, make sure the GitLab `git` account is still allowed.

3. **SSH crypto requirements**: Older SHA-1 RSA keys will stop working, so developers may need new keys before cutover. Ed25519 keys also will not work on FIPS-enabled hosts.

4. **`/tmp` noexec**: This can break Jira/Confluence Java libraries and `gitlab-ctl reconfigure`. The apps should use their own temp directories instead.

5. **Audit failure action**: Using `-f 2` can panic the kernel if the audit backlog fills. Recommend `-f 1` instead. **ISSM decision required.**

6. **Audit disk-full action**: Using `HALT` can take the server down if the audit partition fills. Recommend `SYSLOG`, a dedicated 15 GB audit volume, and alerting. **ISSM decision required.**

7. **IPv4 forwarding**: Docker-based GitLab Runners need this enabled for container networking. **Deviation required for Runner hosts only.**

8. **IPv6 hardening**: Harden IPv6 settings like `accept_ra`, redirects, and source routing, but do not disable IPv6 completely since some apps may rely on `::1`.

9. **Separate `/var` filesystem**: Make sure `/var` is sized appropriately. GitLab and Atlassian app data should live on a separate disk so `/var` does not fill unexpectedly.

10. **AIDE**: Avoid scanning high-churn application data because it can cause heavy disk contention. Keep normal system paths such as `/opt` monitored.

11. **Audit rules**: The full audit rule set can add noticeable CPU/IO load and log growth, especially on GitLab and CI systems. Size the audit volume accordingly and watch for dropped records.

12. **FIPS mode**: Atlassian hosts will need a **CAT I deviation** because FIPS mode is not supported. GitLab should use the `gitlab-fips` package.

13. **Default umask `077`**: This can break applications that need group-readable files. Set a different umask at the service level where required.

14. **Resource limits**: Increase open-file limits for applications that need it. GitLab should generally have at least `65535`.

15. **Host firewall**: With a default-deny firewall, application ports will need to be explicitly opened as part of deployment.

16. **Time synchronization**: Confirm chrony is actually synced to a reachable authoritative source. Bad time sync can break Kerberos, LDAP, and SAML.

17. **Account lockout/password aging**: Service accounts should be `nologin` and non-expiring so lockout or password aging does not stop the application.

18. **Security patching**: Do not allow unattended application upgrades. Version-lock application packages and patch them during planned maintenance windows.

19. **Post-patch config checks**: Check for `.rpmnew` and `.rpmsave` files after patching so proxy or application configs are not missed or replaced.

20. **GRUB authentication**: After setting the GRUB password, test a reboot right away to make sure the server can still boot normally without manual intervention.



```
'openssl s_client -connect packages.microsoft.com:443 -auth_level 0 </dev/null 2>&1 | sed -n '/Certificate chain/,/Server certificate/p'
```

```
openssl s_client -connect packages.microsoft.com:443 -showcerts -auth_level 0 </dev/null 2>/dev/null \
| awk '/BEGIN CERT/,/END CERT/' > /tmp/chain.pem
```

```
awk 'BEGIN{c=0} /BEGIN CERT/{c++; f="/tmp/c" c ".pem"} {print > f}' /tmp/chain.pem
for f in /tmp/c*.pem; do echo "--- $f"; openssl x509 -in $f -noout -subject -issuer; openssl x509 -in $f -noout -text | grep -E 'Public-Key|Signature Algorithm' | head -2; done
```

