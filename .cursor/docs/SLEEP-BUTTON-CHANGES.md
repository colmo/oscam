# OSCam WebUI: botón Sleep (zap Enigma2 / OpenWebif)

Guía para replicar el botón **Sleep** (icono Zzz) en un fork/proyecto OSCam idéntico.

En **Status → Clients**, junto a Hide y Kill, aparece un tercer icono. Al pulsarlo, OSCam hace un GET HTTP al deco del cliente (OpenWebif) para zappear al canal portada:

```
GET http://<IP_CLIENTE>:<httpsleepport><httpsleepzap>
```

Equivalente al comando manual:

```sh
wget -T 5 -t 3 http://IP_DECO/api/zap?sRef=1%3A0%3A1%3A759C%3A422%3A1%3AC00000%3A0%3A0%3A0%3A -O /dev/null
```

Timeout de conexión/lectura: **5 segundos**.

---

## Resumen del proyecto

**OSCam** (*Open Source Conditional Access Module*) es un card server en C (GPL v3). Intermedia entre clientes (receptores, STB) y lectores de tarjetas. La WebUI está en `module-webif.c` + plantillas en `webif/`.

Las plantillas HTML/SVG se registran en `webif/pages_index.txt` y se embeben en tiempo de compilación (`webif/pages_gen.c`). Cualquier plantilla o icono nuevo **debe** aparecer en ese índice.

Flujo relevante:

1. Status lista clientes (`send_oscam_status()`).
2. Hide = `status.html?hide=<CID>`.
3. Kill = `status.html?action=kill&threadid=<CID>`.
4. **Sleep (nuevo)** = `status.html?action=sleep&threadid=<CID>` → GET al IP del cliente.

El polling en vivo (jQuery) reconstruye filas nuevas en `webif/include/jscript.js`; hay que tocar también ahí.

---

## Archivos afectados

| Archivo | Tipo |
|---------|------|
| `webif/status/status_sleepbutton.html` | **Nuevo** |
| `webif/images/ICSLEE.svg` | **Nuevo** |
| `webif/pages_index.txt` | Modificado |
| `globals.h` | Modificado |
| `oscam-config-global.c` | Modificado |
| `module-webif.c` | Modificado (principal) |
| `webif/config/webif.html` | Modificado |
| `webif/include/jscript.js` | Modificado |
| `webif/api.json/header.json` | Modificado |

Tras editar plantillas, recompilar OSCam para regenerar `webif/pages.c` / `webif/pages.h`.

---

## 1. Icono SVG (nuevo)

**Archivo:** `webif/images/ICSLEE.svg`

Formato idéntico al resto de iconos WebIF: una sola línea `data:image/svg+xml;base64,...`.

Contenido completo:

```
data:image/svg+xml;base64,PHN2ZyB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHZlcnNpb249IjEuMSIgd2lkdGg9IjIycHgiIGhlaWdodD0iMjJweCIgdmlld0JveD0iMCAwIDIyIDIyIiBlbmFibGUtYmFja2dyb3VuZD0ibmV3IDAgMCAyMiAyMiIgeG1sOnNwYWNlPSJwcmVzZXJ2ZSI+CiAgPHJlY3QgZmlsbD0ibm9uZSIgd2lkdGg9IjIyIiBoZWlnaHQ9IjIyIi8+CiAgPHRleHQgZmlsbD0iI0ZGRkZGRiIgZm9udC1mYW1pbHk9IkFyaWFsLHNhbnMtc2VyaWYiIGZvbnQtc2l6ZT0iOSIgZm9udC13ZWlnaHQ9ImJvbGQiIHg9IjEiIHk9IjEyIj5aPC90ZXh0PgogIDx0ZXh0IGZpbGw9IiNGRkZGRkYiIGZvbnQtZmFtaWx5PSJBcmlhbCxzYW5zLXNlcmlmIiBmb250LXNpemU9IjciIGZvbnQtd2VpZ2h0PSJib2xkIiB4PSI3IiB5PSIxMCI+ejwvdGV4dD4KICA8dGV4dCBmaWxsPSIjRkZGRkZGIiBmb250LWZhbWlseT0iQXJpYWwsc2Fucy1zZXJpZiIgZm9udC1zaXplPSI2IiBmb250LXdlaWdodD0iYm9sZCIgeD0iMTEiIHk9IjgiPno8L3RleHQ+Cjwvc3ZnPg==
```

SVG original (antes de base64): tres letras blancas `Z z z` en 22×22 px, mismo estilo que `ICKIL.svg` / `ICHID.svg`.

---

## 2. Plantilla del botón (nuevo)

**Archivo:** `webif/status/status_sleepbutton.html`

```html
<A HREF="status.html?action=sleep&amp;threadid=##CID##" TITLE="Sleep ##TARGET##: ##LBL####CLIENTDESCRIPTION##"><IMG CLASS="icon" SRC="image?i=ICSLEE" ALT="Sleep"></A>
```

Patrón copiado de `webif/status/status_killbutton.html`. Placeholders:

- `##CID##` → puntero del cliente (`%p`)
- `##TARGET##` → `"User"`
- `##LBL##` / `##CLIENTDESCRIPTION##` → tooltip

---

## 3. Registrar plantillas en `webif/pages_index.txt`

Junto a `ICKIL`:

```
ICKIL                         images/ICKIL.svg
ICSLEE                        images/ICSLEE.svg
```

Junto a `STATUSKBUTTON`:

```
STATUSKBUTTON                 status/status_killbutton.html
STATUSSBUTTON                 status/status_sleepbutton.html
```

El identificador `ICSLEE` debe coincidir con `image?i=ICSLEE`. El identificador `STATUSSBUTTON` se usa en C con `tpl_getTpl(vars, "STATUSSBUTTON")`.

---

## 4. Campos de configuración

### 4.1 `globals.h`

Dentro de `struct s_config`, bloque `#ifdef WEBIF`, justo después de `http_emmg_clean`:

```c
	int32_t         http_emmg_clean;
	int32_t         http_sleep_port;
	char            *http_sleep_zap;
#endif
```

### 4.2 `oscam-config-global.c`

En `webif_opts[]`, justo después de `httpemmgclean`:

```c
	DEF_OPT_INT32("httpemmgclean"           , OFS(http_emmg_clean),         -1),
	DEF_OPT_INT32("httpsleepport"           , OFS(http_sleep_port),         80),
	DEF_OPT_STR("httpsleepzap"              , OFS(http_sleep_zap),          "/api/zap?sRef=1%3A0%3A1%3A759C%3A422%3A1%3AC00000%3A0%3A0%3A0%3A"),
#ifdef WEBIF_LIVELOG
```

Valores por defecto:

| Opción `oscam.conf` | Campo C | Default |
|---------------------|---------|---------|
| `httpsleepport` | `cfg.http_sleep_port` | `80` |
| `httpsleepzap` | `cfg.http_sleep_zap` | `/api/zap?sRef=1%3A0%3A1%3A759C%3A422%3A1%3AC00000%3A0%3A0%3A0%3A` |

Si `httpsleepzap` está vacío, el botón **no se muestra**.

`DEF_OPT_STR` ya libera el string en `config_free()`; no hace falta un `free` extra.

### 4.3 `oscam.conf` (runtime)

```ini
[webif]
httpsleepport = 80
httpsleepzap = /api/zap?sRef=1%3A0%3A1%3A759C%3A422%3A1%3AC00000%3A0%3A0%3A0%3A
```

---

## 5. Editor WebIf (`webif/config/webif.html`)

Después de la fila `Path for picons:`:

```html
			<TR><TD><A>Path for picons:</A></TD><TD><input name="httppiconpath" type="text" maxlength="127" value="##HTTPPICONPATH##"></TD></TR>
			<TR><TH COLSPAN="2">Sleep (Enigma2 OpenWebif)</TH></TR>
			<TR><TD><A>Sleep zap port:</A></TD><TD><input name="httpsleepport" class="short" type="text" maxlength="6" value="##HTTPSLEEPPORT##"></TD></TR>
			<TR><TD><A>Sleep zap path:</A></TD><TD><input name="httpsleepzap" type="text" maxlength="255" value="##HTTPSLEEPZAP##"></TD></TR>
			<TR><TH COLSPAN="2">Show/Hide in Status</TH></TR>
```

Los `name=` deben coincidir con las claves de `webif_opts[]` (`httpsleepport`, `httpsleepzap`). `webif_save_config("webif", ...)` ya persiste cualquier token de esa sección.

---

## 6. `module-webif.c` (4 inserciones)

### 6.1 Rellenar el formulario Config → WebIf

En `send_oscam_config_webif()`, justo después de `HTTPOSCAMLABEL`:

```c
	tpl_addVar(vars, TPLADD, "HTTPOSCAMLABEL", cfg.http_oscam_label);
	tpl_printf(vars, TPLADD, "HTTPSLEEPPORT", "%d", cfg.http_sleep_port > 0 ? cfg.http_sleep_port : 80);
	tpl_addVar(vars, TPLADD, "HTTPSLEEPZAP", cfg.http_sleep_zap ? cfg.http_sleep_zap : "");
```

### 6.2 Cliente HTTP GET (antes de `send_oscam_status`)

Insertar **justo antes** de `static char *send_oscam_status(...)`.

Función completa:

```c
#define WEBIF_SLEEP_TIMEOUT 5

static int32_t webif_http_get(IN_ADDR_T ip, int32_t port, const char *path)
{
	struct sockaddr_storage sa;
	socklen_t sa_len;
	int s_domain = PF_INET;
	int fd;

	if(!path || !path[0] || port <= 0 || port > 65535 || !IP_ISSET(ip))
		{ return -1; }

	memset(&sa, 0, sizeof(sa));

#ifdef IPV6SUPPORT
	if(!IN6_IS_ADDR_V4MAPPED(&ip) && !IN6_IS_ADDR_V4COMPAT(&ip))
	{
		struct sockaddr_in6 *sin6 = (struct sockaddr_in6 *)&sa;
		s_domain = PF_INET6;
		sin6->sin6_family = AF_INET6;
		sin6->sin6_port = htons((uint16_t)port);
		sin6->sin6_addr = ip;
		sa_len = sizeof(struct sockaddr_in6);
	}
	else
#endif
	{
		struct sockaddr_in *sin = (struct sockaddr_in *)&sa;
		sin->sin_family = AF_INET;
		sin->sin_port = htons((uint16_t)port);
#ifdef IPV6SUPPORT
		memcpy(&sin->sin_addr.s_addr, &ip.s6_addr[12], 4);
#else
		sin->sin_addr.s_addr = ip;
#endif
		sa_len = sizeof(struct sockaddr_in);
	}

	if((fd = socket(s_domain, SOCK_STREAM, IPPROTO_TCP)) < 0)
		{ return -1; }

	set_socket_priority(fd, cfg.netprio);
	set_nonblock(fd, true);

	if(connect(fd, (struct sockaddr *)&sa, sa_len) == -1)
	{
		if(errno == EINPROGRESS || errno == EALREADY)
		{
			struct pollfd pfd;
			pfd.fd = fd;
			pfd.events = POLLOUT;
			if(poll(&pfd, 1, WEBIF_SLEEP_TIMEOUT * 1000) > 0)
			{
				int err = 0;
				socklen_t l = sizeof(err);
				if(getsockopt(fd, SOL_SOCKET, SO_ERROR, &err, &l) != 0 || err != 0)
				{
					close(fd);
					return -1;
				}
			}
			else
			{
				close(fd);
				return -1;
			}
		}
		else
		{
			close(fd);
			return -1;
		}
	}

	set_nonblock(fd, false);

	char host[64];
	cs_strncpy(host, cs_inet_ntoa(ip), sizeof(host));

	char req[512];
	int32_t req_len = snprintf(req, sizeof(req),
		"GET %s HTTP/1.0\r\n"
		"Host: %s\r\n"
		"Connection: close\r\n"
		"\r\n",
		path, host);
	if(req_len <= 0 || req_len >= (int32_t)sizeof(req))
	{
		close(fd);
		return -1;
	}

	struct timeval tv;
	tv.tv_sec = WEBIF_SLEEP_TIMEOUT;
	tv.tv_usec = 0;
	setsockopt(fd, SOL_SOCKET, SO_SNDTIMEO, &tv, sizeof(tv));
	setsockopt(fd, SOL_SOCKET, SO_RCVTIMEO, &tv, sizeof(tv));

	if((int32_t)send(fd, req, req_len, 0) != req_len)
	{
		close(fd);
		return -1;
	}

	char buf[256];
	while(recv(fd, buf, sizeof(buf), 0) > 0) { ; }

	close(fd);
	return 0;
}
```

Notas:

- IPv4 e IPv6 (`IPV6SUPPORT`).
- Connect no bloqueante + `poll` de 5 s.
- GET HTTP/1.0, sin auth (OpenWebif local típico).
- Drena la respuesta y cierra.
- `cs_inet_ntoa()` usa buffer estático: copiar a `host[]` **antes** de cualquier otro uso.

### 6.3 Handler `action=sleep`

En `send_oscam_status()`, **después** del bloque `action == "kill"` y **antes** de `resetserverstats`:

```c
	if(strcmp(getParam(params, "action"), "sleep") == 0)
	{
		char *cptr = getParam(params, "threadid");
		struct s_client *cl = NULL;
		if(strlen(cptr) > 1)
			{ sscanf(cptr, "%p", (void **)(void *)&cl); }

		if(cl && is_valid_client(cl) && (cl->typ == 'c' || cl->typ == 'm')
			&& cfg.http_sleep_zap && cfg.http_sleep_zap[0] && IP_ISSET(cl->ip))
		{
			int32_t port = cfg.http_sleep_port > 0 ? cfg.http_sleep_port : 80;
			if(webif_http_get(cl->ip, port, cfg.http_sleep_zap) == 0)
			{
				cs_log("Sleep zap sent to %s (%s) by WebIF from %s",
					cl->account ? cl->account->usr : "-",
					cs_inet_ntoa(cl->ip), cs_inet_ntoa(GET_IP()));
			}
			else
			{
				cs_log("Sleep zap to %s (%s) failed (WebIF from %s)",
					cl->account ? cl->account->usr : "-",
					cs_inet_ntoa(cl->ip), cs_inet_ntoa(GET_IP()));
			}
		}
	}
```

Condiciones:

- Cliente válido.
- Tipo `'c'` (user) o `'m'` (monitor). No readers/proxies.
- `httpsleepzap` no vacío.
- IP del cliente definida.

La IP destino es `cl->ip` (columna IP del Status).

### 6.4 Pintar el botón en la fila

Donde se asigna el botón Kill a usuarios:

```c
						if(cl->typ == 'c' || cl->typ == 'm')
						{
							tpl_addVar(vars, TPLADD, "TARGET", "User");
							tpl_addVar(vars, TPLADD, "CSIDX", tpl_getTpl(vars, "STATUSKBUTTON"));
							if(cfg.http_sleep_zap && cfg.http_sleep_zap[0])
								{ tpl_addVar(vars, TPLAPPEND, "CSIDX", tpl_getTpl(vars, "STATUSSBUTTON")); }
						}
```

`CSIDX` va a la columna de iconos de acción (`statuscol1`). `TPLAPPEND` concatena Sleep detrás de Kill.

### 6.5 Flag JSON para el poll

Junto a `PICONENABLED` (respuesta JSON de status):

```c
			tpl_printf(vars, TPLADD, "PICONENABLED", "%d", cfg.http_showpicons?1:0);
			tpl_printf(vars, TPLADD, "SLEEPENABLED", "%d", (cfg.http_sleep_zap && cfg.http_sleep_zap[0]) ? 1 : 0);
```

---

## 7. API JSON (`webif/api.json/header.json`)

Después de `piconenabled`:

```json
    "piconenabled":"##PICONENABLED##",
    "sleepenabled":"##SLEEPENABLED##",
```

El poll en vivo lee `data.oscam.sleepenabled`.

---

## 8. Polling JS (`webif/include/jscript.js`)

En `updateStatuspage()`, dentro del bloque `statuscol1` (icono Kill), **después** del `append` de Kill:

```javascript
			if (!is_nopoll('statuscol1')) {
				$(uid + " > td.statuscol1").append('<a title="' + kill2 + ' ' +
					name1 + ': ' + name3 + (item.desc ? '\n' + item.desc.replace('&#13;', '') : '') +
					kill1 + '"><img class="icon" alt="' + kill2 + 
					'" src="image?i=' + kill3 + '"></img>');
				if ((item.type == 'c' || item.type == 'm') && data.oscam.sleepenabled == "1") {
					$(uid + " > td.statuscol1").append('<a title="Sleep ' +
						name1 + ': ' + name3 + (item.desc ? '\n' + item.desc.replace('&#13;', '') : '') +
						'" href="status.html?action=sleep&threadid=' + item.thid.substring(3, item.thid.length) +
						'"><img class="icon" alt="Sleep" src="image?i=ICSLEE"></img>');
				}
			}
```

Sin esto, las filas añadidas por AJAX no tendrían el botón Sleep.

`item.thid` es `"id_<puntero>"`; se usa `substring(3, ...)` igual que Kill.

---

## 9. Compilar y verificar

```sh
./config.sh --enable WEBIF
make
```

El build regenera `webif/pages.c` / `webif/pages.h` a partir de `pages_index.txt`.

Checklist:

1. Status → Clients: icono Zzz junto a Kill en usuarios (no en readers/proxies).
2. Clic → log `Sleep zap sent to ...` o `Sleep zap to ... failed`.
3. El deco cambia al canal portada.
4. Config → WebIf: sección **Sleep (Enigma2 OpenWebif)**.
5. Vaciar `httpsleepzap` y guardar → el botón desaparece.
6. Polling: un cliente nuevo también muestra el icono.

---

## 10. Orden de aplicación recomendado

1. Crear `webif/images/ICSLEE.svg` y `webif/status/status_sleepbutton.html`.
2. Registrar ambos en `webif/pages_index.txt`.
3. Añadir campos en `globals.h` y `oscam-config-global.c`.
4. Editar `module-webif.c` (las 4 inserciones).
5. UI config + JSON + JS.
6. `make` y probar.

---

## Comportamiento / limitaciones

- Destino = IP del cliente OSCam. Si el deco NAT/VPN no coincide con esa IP, el zap fallará.
- Puerto OpenWebif por defecto 80; si usa 8080, poner `httpsleepport = 8080`.
- Sin autenticación HTTP hacia OpenWebif.
- Un intento (no reintenta 3 veces como `wget -t 3`).
- La petición se hace en el hilo de la WebIF (hasta 5 s de bloqueo).
- No aplica a dvbapi local, readers ni proxies.
- `httpsleepzap` debe ser path+query, no URL completa (`/api/zap?sRef=...`).
)
