# WS107
# Настройки haproxy для отказа устойчивости веб-сайта на app.company.msk на машине FW (Frontend и Backend)
global
defaults
  mode https
  timeout client  5s
  timeout server  5s
  timeout connect 5s
  
frontend MyFronted
  bind 200.100.200.100:443 ssl crt /etc/ssl/certs/APP.pem //Привязка порта к внешнему айпи
  default-backend TransparentBack_http
  http-request redirect scheme https unless { ssl_fc }
  
backend TransparentBack_http
  balance roundrobin
  option tcp-check
  server s1 172.20.30.100:80 check weight 3
  server s2 172.20.30.20:80 check weight 1
  
