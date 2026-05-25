"""
PARCHE PARA replicator.py
=========================
Aplica estos 3 cambios al archivo original. No reemplaces todo el archivo,
solo modifica las secciones indicadas.

CAMBIO 1: En __init__, reemplazar la configuración de la IA
──────────────────────────────────────────────────────────
ANTES (líneas ~35-37):
    self.ai_api_url = os.getenv('AI_API_URL', 'https://apifreellm.com/api/v1/chat')
    self.ai_api_key = os.getenv('AI_API_KEY', '')

DESPUÉS:
    self.anthropic_api_key = os.getenv('ANTHROPIC_API_KEY', '')


CAMBIO 2: Reemplazar el método run_ai_filter completo
──────────────────────────────────────────────────────
Busca el método `async def run_ai_filter(self, text):` y reemplázalo entero con esto:
"""

NUEVO_RUN_AI_FILTER = '''
    async def run_ai_filter(self, text: str) -> str | None:
        """
        Usa la API de Claude (Haiku) para:
          - Traducir al español
          - Reemplazar menciones externas por CLUB 10M
          - Devolver REJECT si el mensaje no es una señal válida
        """
        if not self.anthropic_api_key:
            return None

        system_prompt = (
            "Eres el editor de señales de trading de CLUB 10M.\\n\\n"
            "REGLAS ESTRICTAS:\\n"
            "1. Si el mensaje NO contiene una señal de trading real (par/activo + dirección BUY/SELL + niveles), "
            "responde ÚNICAMENTE con la palabra: REJECT\\n"
            "2. Si contiene una señal válida:\\n"
            "   a. Tradúcelo completamente al español. Preserva: BUY, SELL, TP, SL, nombres de activos y números.\\n"
            "   b. Reemplaza CUALQUIER mención a traders, canales o servicios externos (directa o indirecta) por 'CLUB 10M'.\\n"
            "   c. Elimina links, promociones y contenido no operativo.\\n"
            "   d. Mantén emojis si ya venían en el mensaje.\\n"
            "3. Responde SOLO con el mensaje transformado o con REJECT. Sin explicaciones."
        )

        try:
            import httpx
            headers = {
                "x-api-key": self.anthropic_api_key,
                "anthropic-version": "2023-06-01",
                "content-type": "application/json",
            }
            payload = {
                "model": "claude-haiku-4-5-20251001",
                "max_tokens": 512,
                "system": system_prompt,
                "messages": [{"role": "user", "content": text}],
            }
            async with httpx.AsyncClient() as client:
                response = await client.post(
                    "https://api.anthropic.com/v1/messages",
                    headers=headers,
                    json=payload,
                    timeout=20.0,
                )
            if response.status_code == 200:
                result = response.json()["content"][0]["text"].strip()
                logger.info(f"AI Claude: respuesta recibida ({result[:60]}...)")
                return result
            else:
                logger.error(f"AI Claude: error {response.status_code} - {response.text[:100]}")
                return None
        except Exception as e:
            logger.error(f"AI Claude: excepción - {e}")
            return None
'''

"""
CAMBIO 3: En process_message, reactivar el bloque de IA
─────────────────────────────────────────────────────────
Busca esta sección en process_message (alrededor de la línea que dice
"# USAR SOLO TRADUCCIÓN DEL BOT") y reemplázala:

ANTES:
    # 2. Normalización y Traducción con IA (DESACTIVADO TEMPORALMENTE)
    # ai_text = await self.run_ai_filter(text_filtered)
    # ...
    # USAR SOLO TRADUCCIÓN DEL BOT
    final_text = self.smart_fragment_translation(text_filtered)
    if final_text is None:
        logger.warning(...)
        final_text = text_filtered
    logger.info(f"Pipeline: Traducción interna del bot aplicada (IA desactivada).")

DESPUÉS:
"""

NUEVO_PIPELINE_BLOQUE = '''
        # 2. Normalización y Traducción con IA (Claude Haiku)
        ai_text = await self.run_ai_filter(text_filtered)

        if ai_text and ai_text.strip().upper() == "REJECT":
            logger.info(f"Pipeline: Mensaje {msg_id} RECHAZADO por IA (no es señal válida).")
            return

        if ai_text:
            final_text = ai_text
            logger.info(f"Pipeline: Normalización IA (Claude) aplicada.")
        else:
            # Fallback a traducción manual si la IA falla o no está configurada
            final_text = self.smart_fragment_translation(text_filtered)
            if final_text is None:
                logger.warning(f"Pipeline: Traducción fallida en Msg {msg_id}. Usando texto filtrado como fallback.")
                final_text = text_filtered
            logger.info(f"Pipeline: Fallback a traducción manual aplicada.")
'''

"""
──────────────────────────────────────────────────────────────────────────────
VARIABLE DE ENTORNO A AGREGAR EN .env
──────────────────────────────────────────────────────────────────────────────
Agrega esta línea a tu archivo .env (o donde tengas las demás variables):

    ANTHROPIC_API_KEY=sk-ant-xxxxxxxxxxxxxxxxxxxx

Obtén tu API key en: https://console.anthropic.com

Las variables AI_API_URL y AI_API_KEY ya no se necesitan (puedes dejarlas o borrarlas).

──────────────────────────────────────────────────────────────────────────────
DEPENDENCIA A AGREGAR EN requirements.txt
──────────────────────────────────────────────────────────────────────────────
httpx ya está en el proyecto. Solo asegúrate de que esté en requirements.txt:

    httpx>=0.24.0

No necesitas instalar el SDK de Anthropic — usamos httpx directamente,
igual que el resto del bot.

──────────────────────────────────────────────────────────────────────────────
RESUMEN DE LO QUE CAMBIA
──────────────────────────────────────────────────────────────────────────────
- La IA ahora SÍ corre en cada mensaje (antes estaba comentada)
- Si Claude dice REJECT → el mensaje se descarta silenciosamente
- Si Claude transforma el mensaje → se publica la versión transformada
- Si la API de Claude falla → el bot cae al sistema de traducción manual (como antes)
  → el bot NUNCA se rompe por un fallo de la IA

COSTO ESTIMADO (Claude Haiku):
- ~$0.001 por cada 1000 mensajes procesados
- Con 50-100 mensajes/día → menos de $0.10/mes
"""
