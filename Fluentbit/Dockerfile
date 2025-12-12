# Usar la imagen oficial de AWS para Fluent Bit como base
FROM public.ecr.aws/aws-observability/aws-for-fluent-bit:stable

USER root

# Copiar tu archivo de configuración personalizado al directorio de configuración dentro de la imagen
COPY extra.conf /extra.conf