# SELinux
## Запустить nginx на нестандартном порту 3-мя разными способами.
### Переключатели setsebool
Изменим порт nginx на 7777, перезапусти nginx.service и проверим, что он не запустился:
````
root@localhost vagrant]# nano /etc/nginx/nginx.conf
[root@localhost vagrant]# systemctl restart nginx
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
[root@localhost vagrant]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Thu 2021-10-14 09:19:10 UTC; 37s ago
  Process: 26315 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 26332 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=1/FAILURE)
  Process: 26331 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 26317 (code=exited, status=0/SUCCESS)
 
Oct 14 09:19:10 localhost.localdomain systemd[1]: Stopped The nginx HTTP and reverse proxy server.
Oct 14 09:19:10 localhost.localdomain systemd[1]: Starting The nginx HTTP and reverse proxy server...
Oct 14 09:19:10 localhost.localdomain nginx[26332]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Oct 14 09:19:10 localhost.localdomain nginx[26332]: nginx: [emerg] bind() to 0.0.0.0:7777 failed (13: Permission denied)
Oct 14 09:19:10 localhost.localdomain nginx[26332]: nginx: configuration file /etc/nginx/nginx.conf test failed
Oct 14 09:19:10 localhost.localdomain systemd[1]: nginx.service: control process exited, code=exited status=1
Oct 14 09:19:10 localhost.localdomain systemd[1]: Failed to start The nginx HTTP and reverse proxy server.
Oct 14 09:19:10 localhost.localdomain systemd[1]: Unit nginx.service entered failed state.
Oct 14 09:19:10 localhost.localdomain systemd[1]: nginx.service failed.
````
С помощью утилиты sealert сгенерируем отчет об ошибках из лога SELinux:
````
[root@localhost vagrant]# yum install setroubleshoot
[root@localhost vagrant]# sealert -a /var/log/audit/audit.log
````
В выводе видим причину, пот которой nginx не запустился на порту 7777: :
````
SELinux is preventing /usr/sbin/nginx from name_bind access on the tcp_socket port 7777.
````
Также утилита предлагает решить проблему несколькими способами, один из которых - с помощью переключателя setsebool:
````
If you want to allow nis to enabled
Then you must tell SELinux about this by enabling the 'nis_enabled' boolean.

Do
setsebool -P nis_enabled 1
````
