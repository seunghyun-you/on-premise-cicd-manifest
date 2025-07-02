## VirtualBox Nat Network Port Forwarding Setting

|        Name        | Protocol |  Host IP  | Host Port |          Guest IP           | Guest Port |
| :----------------: | :------: | :-------: | :-------: | :-------------------------: | :--------: |
|  ssh_control_node  |   TCP    | 127.0.0.1 |   8022    |          10.0.0.10          |     22     |
| ssh_worker_node_01 |   TCP    | 127.0.0.1 |   8023    |          10.0.0.11          |     22     |
| ssh_worker_node_02 |   TCP    | 127.0.0.1 |   8024    |          10.0.0.12          |     22     |
| web_argocd_server  |   TCP    | 127.0.0.1 |   10443   | <ARGOCD_SERVER_EXTERNAL_IP> |    443     |
