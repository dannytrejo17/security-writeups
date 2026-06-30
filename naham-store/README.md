# NahamStore — mi write-up (en progreso)

Room de **TryHackMe** (medium). Web app / bug bounty style — muchas vulns en una tienda online.  
Hosts: `nahamstore.thm`, `marketing.nahamstore.thm`, `stock.nahamstore.thm`, `internal-api.nahamstore.thm`…

> **Nota:** este write-up lo voy actualizando mientras avanzo. Llevo hasta XXE; faltan partes del room (RCE, SQLi, flags finales).

Room: https://tryhackme.com/room/nahamstore

---

## Cómo empecé

Primero escaneo de puertos:

```bash
sudo nmap -sS -Pn -p- --min-rate 1000 <IP>
```

![nmap -p- — 22, 80 y 8000](screenshots/01-nmap-full.png)

Salieron **22** (ssh), **80** (http) y **8000** (http-alt). Hostname `nahamstore.thm` → `/etc/hosts`.

Más detalle en esos puertos:

```bash
sudo nmap -sC -sV -p22,80,8000 <IP>
```

![nmap -sC -sV — nginx, cookie session, robots /_admin](screenshots/02-nmap-services.png)

nginx en 80 y 8000. En el 80 la cookie `session` sin `HttpOnly`. En el 8000 el `robots.txt` esconde `/_admin`.

### Subdominios

Enumeré vhosts con **ffuf**:

```bash
ffuf -u "http://nahamstore.thm" \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt \
  -H "HOST: FUZZ.nahamstore.thm" -ac
```

![ffuf — www, shop, marketing, stock](screenshots/03-ffuf-subdomains.png)

Los metí en `/etc/hosts`. De los que más usé:
- **`marketing.nahamstore.thm`** — campañas
- **`stock.nahamstore.thm`** — check stock / API
- **`shop.nahamstore.thm`** — tienda (pedidos, perfil, PDF…)

---

## XSS — reflejado (marketing)

Fui a `marketing.nahamstore.thm`. Hay 2 campañas → **View**.

![marketing — campañas](screenshots/04-marketing-campaigns.png)

En la URL sale un identificador. Lo cambié y salió error.

![cambiar id — error](screenshots/05-error-param.png)

El parámetro es **`error`** y refleja el texto (`Campaign not found`). Probé XSS:

![payload en error](screenshots/06-xss-error-payload.png)

![XSS en marketing — alert](screenshots/07-xss-error-result.png)

---

## XSS — buscador (bypass de escape)

En la tienda el buscador usa **`q`**. El input se refleja en pantalla.

![buscador — reflejo de q](screenshots/08-search-reflect.png)

La web escapa etiquetas HTML para que no ejecutes `<script>` directo.

![etiquetas escapadas](screenshots/09-search-escaped.png)

Pero el valor cae en contexto JavaScript. Escapé con:

```
';alert(1)//
```

![bypass JS — alert(1)](screenshots/10-xss-js-bypass.png)

---

## XSS — return order (textarea)

En **Return Order** hay un formulario dentro de un `<textarea>`.

![formulario return order](screenshots/11-return-order-form.png)

Cerrás el textarea y metés script:

```html
</textarea><script>alert(1)</script>
```

![escapar textarea](screenshots/12-textarea-escape.png)

![XSS en return order](screenshots/13-textarea-xss.png)

---

## XSS — producto (parámetro name)

En `/product?id=1&name=...` el **`name`** se refleja en el `<title>`.

![product — param name en URL](screenshots/14-product-name-param.png)

![name reflejado en title](screenshots/15-product-title-reflect.png)

Payload cerrando el title:

```
</title><Script>alert("xss")</Script>
```

![payload en name](screenshots/16-xss-name-payload.png)

![alert xss en producto](screenshots/17-xss-name-alert.png)

---

## XSS — almacenado (User-Agent)

Añadí algo al carrito y fui al pago.

![shopping basket](screenshots/18-shopping-basket.png)

Tras pagar, en la confirmación se refleja el **User-Agent**.

![User-Agent reflejado](screenshots/19-ua-reflected.png)

![página de pago — UA visible](screenshots/20-payment-ua-page.png)

Intercepté con Burp y mandé XSS en el header `User-Agent`:

![Burp — payload en User-Agent](screenshots/21-ua-burp-payload.png)

Al volver a la vista → **stored XSS**.

![XSS almacenado](screenshots/22-stored-xss.png)

---

## Open redirect — parámetro `r`

Con **Arjun** encontré otro param aparte de `q`:

```bash
arjun -u http://nahamstore.thm/
```

![Arjun — q y r](screenshots/23-arjun-params.png)

Probé XSS en `r` sin suerte. Open redirect:

```
http://nahamstore.thm/?r=https://www.google.com
```

![open redirect](screenshots/24-open-redirect.png)

---

## CSRF — cambio de contraseña

En **Change Account Password** intercepté la petición.

![formulario cambiar password](screenshots/25-change-password.png)

Sin token anti-CSRF — el navegador manda las cookies solas.

![Burp — POST sin CSRF token](screenshots/26-csrf-request.png)

Armé un `csrf.html` con POST al endpoint. Lo serví con `python3 -m http.server` y abrí `http://TU-IP/csrf.html`.

![CSRF ejecutado](screenshots/27-csrf-result.png)

---

## IDOR — perfil de usuario

Click en tu usuario → interceptar en Burp.

![perfil en Burp](screenshots/28-profile-burp.png)

Hay un **`id`** en la petición. Cambiándolo ves datos de otro usuario.

![IDOR — cambiar id](screenshots/29-idor-profile.png)

---

## IDOR — generar PDF

En **generar PDF** también va un id.

![PDF — id en request](screenshots/30-pdf-request.png)

El servidor acepta **`user_id`** y confía en el cliente.

![Burp — user_id](screenshots/31-pdf-user-id.png)

Cambiando `user_id` → PDF del pedido de otra persona.

![PDF — order ajeno](screenshots/32-pdf-idor.png)

---

## SSRF — Check Stock

En un producto, **Check Stock** pega a `stock.nahamstore.thm`.

![Check Stock](screenshots/33-check-stock.png)

![petición a stock.nahamstore.thm](screenshots/34-stock-subdomain.png)

`POST /stockcheck` — el parámetro **`server`** controla el destino.

Solo IP → error. Comprueba que “tenga” el dominio pero no valida bien. Con **`@`**:

```
product_id=2&server=stock.nahamstore.thm@192.168.142.92
```

![SSRF — bypass @](screenshots/35-ssrf-at-bypass.png)

Levanté servidor y me llegó la petición.

![callback SSRF](screenshots/36-ssrf-callback.png)

---

## SSRF — internal-api y /orders

Fuzzee rutas internas:

```
server=stock.nahamstore.thm@nahamstore.thm#
```

![SSRF fuzz](screenshots/37-ssrf-fuzz.png)

Encontré `internal-api.nahamstore.thm`:

```
server=stock.nahamstore.thm@internal-api.nahamstore.thm#
```

→ `{"endpoints":["/orders"]}`

![internal-api /orders](screenshots/38-internal-api.png)

Con un order id en la ruta saqué PII + tarjeta:

```
server=stock.nahamstore.thm@internal-api.nahamstore.thm/orders/<order_id>#
```

![SSRF — Jimmy Jones](screenshots/39-ssrf-orders.png)

---

## XXE — donde voy ahora (WIP)

`POST /product/1` en stock pide header **`X-Token`**.

![Missing X-Token](screenshots/40-x-token-required.png)

Arjun encontró el parámetro **`xml`**:

```bash
arjun -u http://stock.nahamstore.thm/product/1
```

![Arjun — xml](screenshots/41-arjun-xml.png)

XML en el body — el servidor refleja el valor en el error:

```xml
<?xml version="1.0"?>
<stockCheck>
    <X-Token>1342</X-Token>
</stockCheck>
```

![XXE — reflejo en error](screenshots/42-xxe-probe.png)

*Sigo desde aquí — inbound/OOB XXE, RCE y SQLi pendientes.*

---

## Mi cadena hasta ahora (resumen)

```
nmap -p- (22, 80, 8000) → nahamstore.thm en /etc/hosts
  → ffuf subdominios → marketing / shop / stock
  → XSS (marketing, q, textarea, name, stored UA)
  → Arjun → open redirect (r)
  → CSRF password
  → IDOR perfil + PDF (user_id)
  → SSRF Check Stock (@) → internal-api/orders
  → XXE en stock (en progreso)
```

---

## Qué uso en este room

- Burp (casi todo)
- ffuf (vhosts)
- Arjun (`r`, `xml`)
- `python3 -m http.server` (CSRF + SSRF callback)

---

## Si lo arreglaran

- Escapar output según contexto (HTML, JS, atributo, title)
- Token CSRF en acciones sensibles
- Autorizar `id` / `user_id` en servidor
- SSRF: validar URL de verdad
- Desactivar entidades externas en XML

---

*Room educativo de TryHackMe. Write-up en progreso — lo actualizo al terminar.*
