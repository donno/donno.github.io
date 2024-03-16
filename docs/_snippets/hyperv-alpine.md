# Setting up Alpine Linux in Hyper-V

## Set-up

1. `setup-alpine`
2. Follow the prompts

## Post reboot

1. Install nano & tmux, `apk update && apk add nano tmux`
2. Modify the message of the day (really the sign-in message).
   `nano /etc/motd`
   * Remove mention of the setup-alpine
   * Remove the mention of editing the message
   * Typically I change it to mention the purpose of the VM. Is it for
     building / testing software or VPN etc.
3. Install Hyper-V guest services
   `apk add hvtools`
4. Set-up Hyper-V guest services, such that they start on boot.
   ```
    rc-update add hv_fcopy_daemon
    rc-update add hv_kvp_daemon
    rc-update add hv_vss_daemon
   ```
5. Run the Hyper-V guest services now to avoid needing to reboot.
    ```
    rc-service hv_fcopy_daemon start
    rc-service hv_kvp_daemon start
    rc-service hv_vss_daemon start
    ```

