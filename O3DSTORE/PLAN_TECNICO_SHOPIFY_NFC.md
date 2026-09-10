# Plan técnico: Shopify + fichas NFC individuales

## Estado actual

La ficha de prueba de O.3D.STORE usa datos ficticios y funciona de forma autónoma.

La activación automática, los botones de contacto reales y el envío de geolocalización todavía **no están implementados**.

## Funcionamiento previsto

1. El cliente compra una placa NFC en Shopify.
2. Introduce los datos de su mascota mediante un formulario.
3. O.3D.STORE crea una ficha individual para esa mascota.
4. La placa NFC se graba con una dirección única, por ejemplo:

```text
https://dominio-de-la-tienda/p/7e2a9c
```

5. Al escanearla, la persona verá únicamente la información autorizada por la familia.
6. El propietario podrá activar, modificar o desactivar la ficha desde el panel privado.

## Datos personalizados en Shopify

La aplicación de O.3D.STORE creará un perfil NFC para cada mascota y lo vinculará al pedido correspondiente.

Primero se definen los campos en `shopify.app.toml`:

```toml
[metaobjects.app.nfc_pet_profile]
name = "NFC pet profile"
display_name_field = "pet_name"
access.admin = "merchant_read_write"

[metaobjects.app.nfc_pet_profile.fields.pet_name]
name = "Pet name"
type = "single_line_text_field"
required = true

[metaobjects.app.nfc_pet_profile.fields.public_id]
name = "Public NFC identifier"
type = "single_line_text_field"
required = true

[metaobjects.app.nfc_pet_profile.fields.profile]
name = "Approved profile data"
type = "json"
required = true

[metaobjects.app.nfc_pet_profile.fields.status]
name = "Activation status"
type = "single_line_text_field"
required = true

[order.metafields.app.nfc_profile]
type = "metaobject_reference<$app:nfc_pet_profile>"
name = "NFC pet profile"
access.admin = "merchant_read_write"
```

## Creación del perfil

Cuando se valide un pedido, la aplicación creará el perfil NFC con un identificador público aleatorio y no secuencial.

```graphql
mutation CreateNfcProfile {
  metaobjectUpsert(
    handle: {type: "$app:nfc_pet_profile", handle: "nfc_7e2a9c"}
    metaobject: {fields: [
      {key: "pet_name", value: "Luna"}
      {key: "public_id", value: "7e2a9c"}
      {key: "profile", value: "{\"contactPhone\":\"+34600000000\",\"sociability\":\"Sociable\"}"}
      {key: "status", value: "pending_activation"}
    ]}
  ) {
    metaobject { id handle }
    userErrors { field message }
  }
}
```

Después, la aplicación enlazará ese perfil con el pedido:

```graphql
mutation LinkOrderToNfcProfile {
  metafieldsSet(metafields: [{
    ownerId: "gid://shopify/Order/1234"
    key: "nfc_profile"
    type: "metaobject_reference"
    value: "gid://shopify/Metaobject/1234"
  }]) {
    userErrors { field message }
  }
}
```

Los identificadores y teléfonos de los ejemplos son ficticios.

## Lectura segura de la ficha

El panel privado puede recuperar el vínculo con el perfil desde el pedido:

```graphql
query LoadNfcProfileForOrder {
  order(id: "gid://shopify/Order/1234") {
    nfcProfile: metafield(key: "nfc_profile") {
      jsonValue
    }
  }
}
```

La ficha pública no debe consultar Shopify directamente desde el navegador. La dirección NFC debe abrir una ruta propia de O.3D.STORE, que mostrará solamente los datos que la familia haya autorizado.

## Privacidad y seguridad

- El identificador NFC debe ser aleatorio y difícil de adivinar.
- La ficha pública solo mostrará información autorizada.
- Los datos completos permanecerán en el sistema privado.
- Una ficha desactivada dejará de mostrar los datos públicos.
- La aplicación comprobará que cada pedido cree solo un perfil por mascota.

## Fases pendientes

1. Crear la aplicación privada de O.3D.STORE.
2. Añadir el formulario para los datos de la mascota.
3. Conectar el pedido de Shopify con la creación de la ficha.
4. Crear el panel privado de activación y edición.
5. Grabar y probar una etiqueta NFC real.
6. Activar llamadas, SMS y WhatsApp solo tras las pruebas.
7. Como mejora futura, pedir permiso de ubicación a quien encuentre la mascota y enviar esa posición mediante un sistema seguro.

La geolocalización no debe anunciarse como disponible hasta que se haya implementado, cuente con consentimiento y haya sido probada en móviles reales.
