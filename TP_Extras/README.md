# TP - Extras

## Load Balancing

Modif de httpd.conf :

```txt
<Location "/balancer-manager"> # Permet d'avoir un accès à une page de monitoring pour le load balancing (host/balancer-manger)
    SetHandler balancer-manager
    Require all granted
</Location>

<Proxy "balancer://myApi"> # On déclare les workers qui vont faire le load balancing
    BalancerMember "http://api_1:8080"
    BalancerMember "http://api_2:8080"
    ProxySet lbmethod=bytraffic # Algorithme de planification avec répartition de charge en fonction d'un niveau de trafic pour le module (?)
</Proxy>

ServerName localhost
<VirtualHost *:80> # On déclare les "routes" ici. Les lignes avec le "!" permettent d'indiquer qu'on ne veut pas passer par le proxy pour cette ressource en particulier.
    ProxyPreserveHost On
    ProxyPass /api balancer://myApi
    ProxyPassReverse /api balancer://myApi
    ProxyPass /balancer-manager !
    ProxyPassReverse /balancer-manager !
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