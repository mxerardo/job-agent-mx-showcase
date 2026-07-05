# 🔍 Job Agent MX

Agente automatizado que busca vacantes de empleo en Computrabajo las evalúa
con IA contra perfiles profesionales, y notifica los mejores matches por Telegram.
Corre 2 veces al día de forma completamente automática, sin servidor propio.

> **Nota:** el código fuente de este proyecto es privado. Este repositorio documenta
> la arquitectura, las decisiones técnicas y el funcionamiento real del sistema.
> Con gusto comparto acceso al código completo bajo solicitud (ver [Contacto](#contacto)).

---

## Por qué lo construí

Durante mi propia búsqueda de empleo me di cuenta de que revisar manualmente el
portal todos los días, para varios perfiles distintos (dada mi experiencia), era 
lento y repetitivo, exactamente el tipo de problema que la automatización debería resolver. 
Construí este agente para que hiciera ese trabajo por mí: buscar, filtrar, evaluar 
relevancia con IA, y avisarme solo cuando algo realmente vale la pena revisar.

## Cómo funciona

```
GitHub Actions (cron 8am/6pm) 
    → Scraper (Computrabajo)
    → Filtro por ubicación (Guanajuato / Querétaro)
    → Filtro por sueldo mínimo
    → Deduplicación (fingerprint MD5, historial de 14 días)
    → Evaluación con LLM (match contra perfil profesional, 0-100%)
    → Notificación por Telegram (si supera el umbral del perfil)
    → Historial persistente (commit automático a Git)
```

El sistema evalúa cada vacante contra **tres perfiles profesionales distintos** en
cada corrida (ventas, compras/abastecimiento, y desarrollo de software), cada uno
con su propio umbral de relevancia y filtros de sueldo.

## Decisiones técnicas destacadas

- **Deduplicación vía fingerprint MD5.** Cada vacante se identifica con un hash de
  `título + empresa + ubicación` normalizado. El historial se guarda en el propio
  repo (`data/history.json`) con retención de 14 días, y se actualiza con un commit
  automático después de cada corrida — sin necesidad de una base de datos externa.

- **Regex en vez de selectores CSS para extraer el sueldo.** Los selectores CSS de
  los portales cambian con cada rediseño; extraer el dato del texto plano de la
  tarjeta de vacante con expresiones regulares resultó más resistente a esos cambios.

- **Filtro de sueldo con "beneficio de la duda".** Las vacantes que no especifican
  sueldo *pasan* el filtro en vez de descartarse — solo se excluyen las que ofrecen
  explícitamente menos del mínimo. Decisión deliberada para no perder oportunidades
  por falta de dato, algo común en el portal de Computrabajo.

- **Migración a DeepSeek API.** El proyecto empezó usando la API de Claude y migró
  a DeepSeek (endpoint compatible con OpenAI) por reducción de costo en tokens y soporte
  de caching, manteniendo la misma lógica de evaluación.

- **Infraestructura 100% gratuita.** Todo corre sobre GitHub Actions como scheduler,
  sin servidor propio, una restricción de diseño deliberada, no una limitación
  técnica.

- **Manejo de errores tolerante en scraping.** Los scrapers usan bloques try/except
  amplios para que una tarjeta de vacante mal formada no interrumpa la corrida
  completa — priorizando robustez sobre precisión en el manejo de excepciones.

## Código destacado

Algunos fragmentos representativos de las decisiones técnicas mencionadas arriba.
El código completo (scrapers, prompts de evaluación, configuración de perfiles) se
mantiene privado.

**Extracción de sueldo por regex, con validación de rango:**

```python
@staticmethod
def parse_salary(texto):
    """Extrae el valor numérico mínimo del texto de sueldo"""
    if not texto:
        return None

    texto_lower = texto.lower().strip()

    # Descartar textos no concretos
    if any(p in texto_lower for p in ['convenir', 'negociable', 'atractivo', 'competitivo', 'no especif']):
        return None

    # Busca números con separadores de miles: $20,000 / $20.000 / 20000
    matches = re.findall(r'\d{1,3}(?:[,\.]\d{3})+|\d{4,6}', texto)

    valores = []
    for m in matches:
        n = re.sub(r'[,\.]', '', m)
        try:
            val = int(n)
            if 3000 <= val <= 300000:  # Rango razonable MXN
                valores.append(val)
        except:
            continue

    return min(valores) if valores else None
```

**Fingerprint MD5 para deduplicación, con manejo del caso "empresa no disponible":**

```python
@staticmethod
def compute_fingerprint(job):
    """Genera MD5 de title + company + location normalizados"""
    title    = (job.get('title') or '').lower().strip()
    company  = (job.get('company') or '').lower().strip()
    location = (job.get('location') or '').lower().strip()
    # Si company es N/A, no incluirlo en el fingerprint
    if company in ('n/a', ''):
        key = f"{title}|{location}"
    else:
        key = f"{title}|{company}|{location}"
    return hashlib.md5(key.encode('utf-8')).hexdigest()
```

**Deduplicación contra historial persistente, con rotación automática por antigüedad:**

```python
def deduplicate(self, jobs, history_path='../data/history.json', retention_days=14):
    """Filtra jobs cuyo fingerprint ya está en historial. Retorna (new_jobs, stats)"""
    history_path = Path(history_path)
    history_path.parent.mkdir(parents=True, exist_ok=True)

    try:
        with open(history_path, 'r', encoding='utf-8') as f:
            history = json.load(f)
    except (FileNotFoundError, json.JSONDecodeError):
        history = {'fingerprints': {}, 'metadata': {'retention_days': retention_days}}

    fingerprints = history.get('fingerprints', {})
    now = datetime.now()

    new_jobs = []
    duplicates = 0
    for job in jobs:
        fp = self.compute_fingerprint(job)
        if fp in fingerprints:
            duplicates += 1
        else:
            new_jobs.append(job)
            fingerprints[fp] = now.isoformat()

    # Rotar fingerprints antiguos (> retention_days)
    cutoff = now.timestamp() - (retention_days * 24 * 3600)
    rotated = 0
    for fp, timestamp_str in list(fingerprints.items()):
        try:
            ts = datetime.fromisoformat(timestamp_str).timestamp()
            if ts < cutoff:
                del fingerprints[fp]
                rotated += 1
        except:
            continue

    history['fingerprints'] = fingerprints
    history['metadata']['last_updated'] = now.isoformat()

    with open(history_path, 'w', encoding='utf-8') as f:
        json.dump(history, f, indent=2, ensure_ascii=False)

    return new_jobs, {'new': len(new_jobs), 'duplicates': duplicates,
                       'total_in_history': len(fingerprints), 'rotated': rotated}
```

## Stack técnico

![Python](https://img.shields.io/badge/Python-3776AB?style=flat&logo=python&logoColor=white)
![GitHub Actions](https://img.shields.io/badge/GitHub_Actions-2088FF?style=flat&logo=githubactions&logoColor=white)
![OpenAI SDK](https://img.shields.io/badge/OpenAI_SDK-412991?style=flat&logo=openai&logoColor=white)
![Telegram Bot API](https://img.shields.io/badge/Telegram_Bot_API-26A5E4?style=flat&logo=telegram&logoColor=white)
![Google Sheets API](https://img.shields.io/badge/Google_Sheets_API-34A853?style=flat&logo=googlesheets&logoColor=white)

## Resultado

<img src="https://github.com/user-attachments/assets/3205f63f-686c-49ec-85fc-be920b9fe9f6" alt="bot_1" style="width: 400px; height: auto;" />

## Contacto

¿Te interesa ver el código completo o conversar sobre el proyecto?

[LinkedIn](www.linkedin.com/in/ingerardo) · [Email](gerardo.garciaj@gmail.com)
