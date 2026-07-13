# Usklađenost arhitekture s ciljevima projekta

Zadatak traži 
# demonstraciju sigurne isporuke, 
# upravljanja slikama, orkestracije,
# observability-ja i troubleshootinga.

Odabrana arhitektura izravno podržava sve navedeno:

# Sigurna isporuka: 
    svaki servis je zaseban kontejner s vlastitim, minimalnim opsegom ovisnosti — manja površina za napad po servisu, mogućnost zasebnog skeniranja i hardeninga svake slike.
# Observability i troubleshooting: 
    API izlaže /healthz i /readyz endpointe koji se koriste kao liveness/readiness probe u Kubernetesu i kao dijagnostički alat u incidentnim scenarijima.
# Orkestracija: 
    decoupling (razdvajanje) API/Worker preko Redis reda omogućuje neovisno skaliranje i rolling update pojedinih komponenti bez gubitka narudžbi.  
# Upravljanje konfiguracijom i tajnama: 
    svaka komponenta prima konfiguraciju isključivo kroz environment varijable (.env lokalno, ConfigMap/Secret u produkciji), bez hardkodiranih vrijednosti u kodu.