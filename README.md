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

