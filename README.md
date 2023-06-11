# alertmanager-webhook
If you are using Alert Manager to notify you of events and you have a rule for high RAM load, you can use alert manager to trigger an action to restart the node (or clear the cache) to free up memory whenever it exceeds a certain threshold.

I have this rule for high ram load:
```
  - alert: HostHighRamLoad
    expr: (node_memory_MemTotal_bytes - node_memory_MemFree_bytes - (node_memory_Cached_bytes + node_memory_Buffers_bytes)) / node_memory_MemTotal_bytes *100 > 80
    for: 0m
    labels:
      severity: warning
    annotations:
      summary: Host high RAM usage
      description: "RAM usage is > 80%\n  VALUE = {{ $value }}"
```
which triggers whenever the RAM usage exceeds 80%. I'm going to use this rule to trigger a restart of the node.

## Steps
1. Install [webhook](https://github.com/adnanh/webhook) with:
`sudo apt-get install webhook`.
There are other methods to install it mentioned in the repo, but I'll assume you're using the Ubuntu package manager.
2. Create a script that holds the command to restart the node:
`nano restart-node.sh`
The content of the script should be:
```
#!/bin/bash
<COMMAND TO RESTART THE NODE OR THE CONTAINER OR TO CLEAR THE CACHE>
```
3. Make it executable:
`chmod +x restart-node.sh`
4. Create the webhook YAML file (we'll call it `restart-hook.yml` and it's located in `/home/ubuntu`):
```
- id: restart-node
  execute-command: "/home/ubuntu/restart-node.sh"
  command-working-directory: "/home/ubuntu"
```
5. Create a service that will be running the webhook to listen for events:
`sudo nano /etc/systemd/system/restart-webhook.service`
It should look like this:
```
[Unit]
Description=Restart Node Webhook

[Service]
User=ubuntu
Group=ubuntu
ExecStart=webhook -verbose -debug -ip localhost -hooks /home/ubuntu/restart-hook.yml
Restart=on-failure

[Install]
WantedBy=multi-user.target
```
This will run the webhook and listen for our events on `http://localhost:9000/hooks/restart-node`. If you want to change the port you can add the flag `-port <PORT>`.
6. Start the service:
```
sudo systemctl daemon-reload
sudo systemctl enable restart-webhook
sudo systemctl start restart-webhook
sudo systemctl status restart-webhook
```
7. Now it's time to add the proper triggers in Alert Manager. Open the manager's yaml file and add the following triggers (`...` are other rules you might have):
```
route:
 ...
 
 routes:
 - matchers:
   - alertname="HostHighRamLoad"
   receiver: "webhook"

receivers:
...

- name: 'webhook'
  webhook_configs:
  - url: 'http://localhost:9000/hooks/restart-node'
```
8. Save the file and restart alert manager. Now whenever RAM usage exceeds 80% the script will run and trigger the specified action!
