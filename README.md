# Personal-Wifi-Zone

Fibertel/Flow/Personal o como se llame implementó un "servicio" llamado Personal Wifi Zone. Este servicio lo que hace es usar los modem/router que les dan a los clientes para ofrecer servicio de internet a cualquiera otro cliente que pasa por la calle. Esencialmente si alguien está en la calle y es cliente de Fibertel se puede conectar a esa red (en realidad cualquiera se puede conectar porque está abierta, lo que pasa es que luego para poder usar internet necesita loguearse en Fibertel/Personal/Flow).


Esto lo hicieron compulsivamente, sin avisar a los clientes que cualquiera que pasa por la puerta de tu casa va a usar el aparato que tenes en comodato y tu energía eléctrica para ofrecerles internet. La falta de ética de esta empresa no es novedad pero lo más intrigante es que resulta casi imposible apagar este servicio.


En el transcurso del último mes, hablé con aproximadamente 10 personas de atención al cliente. Al principio negaban que el modem/router era el que daba el servicio y decían que era un servicio que ofrecían desde los postes de internet. Luego de intentar convencerlos de que no era así, algunas de las personas de atención al cliente lo admitían y decían que no se podían apagar. Luego, de hablar con varios, uno consiguió elevar un ticket al sector técnico y lograron apagarlo en mi aparato. Esto duró algo así como 4 días, luego volvió a parecer, supongo que con un update del firmware del modem/router.


Creo que este servicio debería ser optativo, cuanto menos deberían avisar a los clientes que con nuestro consumo eléctrico (por más que sea menor) vamos permitir que Fibertel ofrezca más servicios a otros clientes. Como mínimo creo que debería ser fácil de dar de baja, pero no.


Dicho esto, estuve buscando una forma de sacarlo de mi aparato

## ¿Cómo deshacernos de Personal Wifi Zone?

### Estrategia 1:

Poner el modem/router en modo bridge y poner un router propio. Esto anda pero necesitamos otro aparato.


### Estrategia 2: 

Apagar la red que funciona en 2.4Ghz, esto hace que se apague la red Personal Wifi Zone también porque usa la misma antenena se ve. El problema de esto es que nos quedamos sin red 2.4 Ghz


### Estrategia 3:
Apagarlo en la configuración del modem/router.

**[spoiler]** No conseguí que desaparezca, pero si que sea inutil, que no ande.


Entrando a http://192.168.0.1 (en mi caso, podría variar) podemos entrar en la configuración del aparato. En mi caso tenía los valores por default (u:custadmin, p:cga4233) que se pueden cambiar (y conviene hacerlo!). El problema es que dentro de la configuración no encontré ninguna forma de apagar esta red (debe ser pq no quieren :p). Pero viendo las request que hace cuando navegamos encontré que la red estaba ahí.


A continuación listo como encontrarla usando Python. Lo primero que hice fue loguearme desde un browser y copiarme las credenciales para no implementar el login. Esto lo hacemos con el inspect en Chrome, en Applications/Cookies/http:192.168.0.1 y la guardo en un diccionario

```python
import requests

cookies = {
    'auth': '<COMPLETAR>',
    'PHPSESSID': '<COMPLETAR>',
}

headers = {
    'Accept': '*/*',
    'Accept-Language': 'es-419,es',
    'Connection': 'keep-alive',
    'Referer': 'https://192.168.0.1/',
    'User-Agent': 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/107.0.0.0 Safari/537.36',
    'X-CSRF-TOKEN': cookies['auth'],
    'X-Requested-With': 'XMLHttpRequest',
}

```

Luego me fijo cual es el id de la red. El mio era el 9, itere al principio hasta 5 y luego fui agregando hasta que lo encontre 

```python

ids_devices_list = list(map(str, range(1,10)))
res = requests.get(f'https://192.168.0.1/api/v1/wifi/{",".join(ids_devices_list)}/SSIDEnable,SSID,BSSID,SSIDAdvertisementEnabled', cookies=cookies, headers=headers, verify=False).json()
df = pd.DataFrame([res[i]['data'] for i in ids_devices_list if i in res])

```

esto da


```python
  SSIDEnable                SSID              BSSID SSIDAdvertisementEnabled
       true            mired     16:78:68:AB:F0:C8                     true
       true           mired5ghz  16:78:68:AB:F0:D0                     true
      false      Tch-Guest3-2.4  0A:78:68:AB:F0:C9                    false
      false        Tch-Guest4-5  0A:78:68:AB:F0:D1                    false
      false      Tch-Guest5-2.4  0A:78:68:AB:F0:CA                     true
      false        Tch-Guest6-5  0A:78:68:AB:F0:D2                     true
      false      Tch-Guest7-2.4  0A:78:68:AB:F0:CB                     true
      false        Tch-Guest8-5  0A:78:68:AB:F0:D3                     true
       true  Personal Wifi Zone  0A:78:68:AB:F0:CC                     true

```


Ahi estaba, en la 9!


Lo que hice a continuación fue fijarme que http request se hacian cuando apagamos y prendemos redes pero no funciono con la Personal Wifi Zone, me decia que esta red no implementaba esos mensajes o algo asi (si me anduvo con las mias). Por lo que se me ocurrio probar habilitarle el mac filtering. De esta manera la red seguiría abierta pero solo podrian conectarse de dispositivos con mac address de la lista (que estaba vacía). Ese mensaje si se lo banco.


Para hacerlo, osea para forzara que use  hay que:


```python
net_id = '9'
data = {
    'ACLEnable': 'true',
    'FilterAsBlackList': 'false',
    'ACLTbl[0][__id]': '0',
}

response = requests.post('https://192.168.0.1/api/v1/wifi/'+net_id, cookies=cookies, headers=headers, data=data, verify=False)

```

esto lo que logró fue que ahora si alguien queria conectarse a la red, podía hacerlo pero no iba tener IP valida, ni por consiguiente llegar a internet. No soluciona el problema pero ayuda mientras los queridos amigos de Fibertel resuelven esto.


Para volver habilitarla basta ejecutar
```python
data = {
    'ACLEnable': 'false',
    'FilterAsBlackList': 'false',
    'ACLTbl[0][__id]': '0',
}

response = requests.post('https://192.168.0.1/api/v1/wifi/9', cookies=cookies, headers=headers, data=data, verify=False)
```


### Disclaimer
No soy para nada experto en esto, seguro que existen mejores soluciones, por favor compartanla si tienen una mejor!

