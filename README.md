# Tukicards

Sitio estático de gift cards por país. Se despliega en Vercel **desde la raíz
del repositorio**: lo que ves en la raíz es exactamente lo que se sirve.

## Cómo trabajar

1. Editás los CSV de `sheets/` (catálogo, países, planes) y el copy de
   `build/contenido.js`.
2. Corrés el build:

       node build/build.js

   El build **publica directo en la raíz**. No hay que copiar nada a mano.
   Deja constancia de lo que generó en `.build-manifest.json`, y en la corrida
   siguiente borra solo los archivos que él mismo había creado y ya no
   corresponden. Nunca toca `img/`, `styles.css`, `pagina.css`, `checkout/`
   ni `favicon.svg`.
3. Verificás antes de subir:

       python3 work/verificar.py     # schema, títulos, tramos G2A, registro
       python3 work/probar-urls.py   # imita Vercel y pide todas las URLs

4. Commit y push. Vercel publica la raíz tal cual.

> **Importante:** el build falla a propósito si intenta escribir sobre una
> carpeta "intocable". Si alguna vez ves páginas de país en 404 o imágenes que
> desaparecen, casi seguro se copió el proyecto un nivel más adentro de lo que
> debía: la raíz del deploy tiene que ser la carpeta que contiene `index.html`
> y `vercel.json`, no una carpeta que contenga a esa.

## Estructura

| Carpeta | Qué es | ¿Lo genera el build? |
|---|---|---|
| `index.html`, `pe/`, `pa/`, `*-gift-card/`, institucionales | páginas | sí |
| `sitemap.xml`, `robots.txt`, `data/` | metadatos | sí |
| `img/` | imágenes (tarjetas, banderas, hero) | no — `work/componer.py` |
| `styles.css`, `pagina.css`, `favicon.svg` | estáticos | no |
| `checkout/` | checkout (HTML+JS a mano) | no |
| `sheets/` | catálogo, países, planes | no |
| `build/` | el generador | no |
| `work/` | scripts auxiliares y originales de imágenes | no |

## Imágenes

`python3 work/componer.py` compone `img/cards/{marca}-{iso}.webp` a partir de
las originales de `work/originales/`, agregando el fondo de marca y la chapa
con la bandera del país.

## Analítica

Las páginas cargan Vercel **Web Analytics** y **Speed Insights** desde
`/_vercel/insights/script.js` y `/_vercel/speed-insights/script.js`. Esos
archivos los inyecta la plataforma, no están en el repo: **en local dan 404 y
es normal**, por eso los verificadores los saltean.

Para que empiecen a registrar hay que habilitarlos una vez en Vercel:
proyecto → pestaña **Analytics** → *Enable*, y lo mismo en **Speed Insights**.
Sin ese paso los scripts se cargan pero no reportan nada.

Son sin cookies, así que no hacen falta banners de consentimiento.

El checkout además dispara un evento propio `pedido_enviado` (con marca, país y
monto) cuando alguien manda el formulario. Sirve para medir el embudo real:
visitas a la ficha → checkout → pedido enviado. Los eventos personalizados
dependen del plan; si no están disponibles, la llamada no hace nada y el
checkout funciona igual.

## Reglas que el build hace cumplir

- La similitud se informa para revisión editorial; no elimina variantes regionales solicitadas.
- Precios de planes con más de 45 días no se usan para calcular meses. Las fichas conservan saldo, precio y compra.
- El catálogo y las referencias se validan antes de escribir los archivos publicados.
- Cada denominación tiene que mapear a un tramo de Crypto Voucher en G2A
  (10/15/20/25/30/35/40/45/50/60 USD), que es lo que el checkout le pide al cliente.
- Los hubs de marca salen del `noindex` recién cuando hay 2+ países.

El checkout asigna el Crypto Voucher más cercano a la referencia `usd` del catálogo. En empate elige el menor. El precio de venta del voucher en G2A puede diferir de su valor nominal.

## Ampliación del catálogo

Siete marcas en diez países: Netflix, Spotify, Google Play, Disney+, PlayStation Plus, Prime Video y HBO Max; PE, PA, AR, CL, CR, VE, EC, BO, BR y MX.

`Catalogo.csv` incluye `Unidad` (`saldo` o `meses`) y `Plan`. En PlayStation Plus el nominal es la duración: 1/3/6 meses de Essential, con vouchers de USD 10/30/60. Disney+ usa saldo y vouchers de USD 10/20/50; no se inventan tarifas oficiales para calcular su duración.

En los ocho países originales, las nuevas opciones usan la escala comercial ya existente de Google Play. Brasil: saldo R$55 / precio R$57 por referencia USD 10. México: saldo MX$200 / precio MX$206 por referencia USD 10. Son precios de venta editables de Tukicards, no cotizaciones cambiarias ni precios oficiales de las marcas. Las denominaciones mayores mantienen esa proporción.

`build/nuevos-contenidos.js` contiene los textos de las 26 altas y `build/ampliacion.js` las fichas de saldo/duración y la portada brasileña. Brasil usa portugués, BRL y `pt-BR`, incluido el checkout. `sheets/Nuevos-productos.csv` registra las altas; la fuente del build sigue siendo `Catalogo.csv`.

### Prime Video y HBO Max — septiembre de 2026

`build/streaming.js` incorpora las definiciones de marca y los 20 artículos nuevos,
con introducciones, guías y preguntas propias por país. Los dos hubs, la navegación,
el catálogo JSON y el sitemap se generan con el resto del sitio. Los slugs son
`prime-video-gift-card` y `hbo-max-gift-card`.

Las tres opciones de cada nueva ficha mantienen la escala comercial de Disney+
del mismo país y corresponden a Crypto Voucher de **USD 10, 20 y 50**. No son
tarifas oficiales ni una equivalencia en meses. El contenido no presupone un
canje universal mediante saldo Amazon ni asigna planes de HBO Max.
`work/ampliar-streaming.py` registra esas altas y sus imágenes SVG de forma
idempotente; no se necesita volver a ejecutarlo para compilar.

La fecha editorial explícita es **2026-09-20**, en `build/templates.js`: se
muestra en las fichas y hubs de marca, en `WebPage.dateModified` y en el sitemap.
No cambia automáticamente al compilar y no altera las fechas de las tarifas
de `Planes.csv`. Las nuevas guías son distintas, pero ningún porcentaje de
similitud garantiza posicionamiento ni inmunidad a las políticas de Google.

### Revisión SEO y checkout

El checkout usa dos pasos: comprar el código en G2A y volver para pegarlo.
`python work/verificar-seo.py` comprueba las 97 URLs indexables, canonical,
metadatos, hreflang recíproco, imágenes sociales y datos estructurados. El informe
de cambios y pendientes está en `work/seo-2026-09-20/INFORME.md`.

Las colecciones de marca usan `CollectionPage`/`ItemList`; las fichas usan
`Product`. El checkout conserva `noindex` y permite rastreo. `priceValidUntil`
no se inventa al compilar. `python work/render-social.py` renderiza los SVG a PNG
para vistas previas sociales cuando se incorporan o modifican imágenes.
