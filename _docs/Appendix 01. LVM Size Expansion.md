#### lvm extend

```bash
lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
```

#### partition exten

```bash
resize2fs /dev/ubuntu-vg/ubuntu-lv
```
