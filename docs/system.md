## 9. System Programming

### POSIX

Compatibility standard for UNIX-like systems. Defines APIs for files, processes, signals, and threads.
Python modules: `os`, `signal`, `threading`, `subprocess`, `socket`.

### Open Ports in Linux

```bash
ss -tlnp        # recommended
netstat -tlnp   # older but widely available
lsof -i -P -n   # list open sockets
```

---
