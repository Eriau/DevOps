# TP - Extras

## Load Balancing

Modif de httpd.conf :

```txt
<Proxy "balancer://myApi">
    BalancerMember "http://api_1:8080"
    BalancerMember "http://api_2:8080"
</Proxy>

ServerName localhost
<VirtualHost *:80>
    ProxyPreserveHost On
    ProxyPass /api balancer://myApi
    ProxyPassReverse /api balancer://myApi
    ProxyPass / http://front:80/
    ProxyPassReverse / http://front:80/
</VirtualHost>
```

Et on active les modules suivants :

```txt
LoadModule proxy_ajp_module modules/mod_proxy_ajp.so
LoadModule proxy_balancer_module modules/mod_proxy_balancer.so
LoadModule slotmem_shm_module modules/mod_slotmem_shm.so
LoadModule lbmethod_byrequests_module modules/mod_lbmethod_byrequests.so
```