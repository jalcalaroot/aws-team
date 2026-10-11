# aws-team - contexto del proyecto

Fork de respaldo de [`aws-samples/iam-identity-center-team`](https://github.com/aws-samples/iam-identity-center-team) (TEAM, acceso elevado temporal sobre IAM Identity Center), fijado en **v1.5.2** (publicada el 2026-09-23, licencia MIT-0). Creado el 2026-10-11 como respaldo para uso futuro. **No está desplegado**; no se ha ejecutado nada en la cuenta de AWS.

## Para qué existe

Es la pieza de "acceso temporal" (equivalente a Entra PIM) de la POC de federación Entra ID -> AWS. Ver `RUNBOOK.md` para prerrequisitos, despliegue, costo y desinstalación. La documentación original está en `docs/docs/`.

## Decisiones y hallazgos que dieron origen al repo (2026-10-08 al 2026-10-11)

- El tenant de Entra (`Directorio predeterminado`, `johan104hotmail.onmicrosoft.com`) **no tiene licencias** (0 SKUs): sin Entra ID P2 no hay PIM. P2 cuesta unos 9-10 USD por usuario al mes y la prueba de 30 días pide una cuenta cloud de Entra, no una cuenta personal de Microsoft.
- Alternativa elegida para el acceso temporal: **TEAM en AWS**, con Entra solo como fuente de identidad (SAML, y SCIM si la licencia lo permite). Así no se necesita P2.
- Para personas, la primera opción evaluada fue SAML directo a IAM (una cuenta, sin Organizations); TEAM exige el camino de Identity Center. Para aplicaciones y pipelines: OIDC con `AssumeRoleWithWebIdentity`.
- **TEAM exige una organización de AWS Organizations**, Identity Center como instancia de organización (la instancia de cuenta no sirve: no tiene permission sets) y una cuenta miembro dedicada como administradora delegada. Con cuentas individuales hay que crear la organización primero.
- TEAM no puede dar acceso temporal a la cuenta de administración ni gestionar permission sets de esa cuenta.
- El costo principal es CloudTrail Lake (ingesta y almacenamiento de eventos de gestión).

## Pendiente antes de desplegar

1. Comprobar si la cuenta `<AWS_ACCOUNT_ID>` ya pertenece a una organización y cuántas cuentas hay en total.
2. Crear la organización, habilitar Identity Center y una cuenta miembro para TEAM.
3. Conectar Entra a Identity Center (SAML; SCIM solo si el tenant lo permite).
4. Estimar el costo de CloudTrail Lake con la calculadora de precios de AWS.
5. Crear el secret de Secrets Manager con un token **classic** de GitHub apuntando a este fork.

## Convenciones

- `origin` = este fork; `upstream` = `aws-samples`. Los archivos propios (`RUNBOOK.md`, `CLAUDE.md`) no existen upstream, así que sincronizar con `upstream/main` no genera conflictos.
- `deployment/parameters.sh` contiene IDs de cuenta y está en `.gitignore`: no se versiona.
