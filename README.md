# 📘 Web Scraper E-Commerce Colombia — Documentación Técnica de Arquitectura

---

| **Proyecto**       | Web Scraper + Análisis IA para E-Commerce Colombiano |
|:-------------------|:----------------------------------------------------|
| **Autor**          | Joan Steven Gonzalez Maldonado                       |
| **Fecha**          | Marzo 2026                                           |
| **Tecnologías**    | Python 3.13 · Selenium · BeautifulSoup · Ollama / LM Studio |
| **Repositorio**    | GitHub — Repo-prueba-técnica                         |

---

## Tabla de Contenido

1. [Introducción al Proyecto](#1-introducción-al-proyecto)
2. [¿Por qué Clean Architecture?](#2-por-qué-clean-architecture)
3. [Estructura del Proyecto y Capas](#3-estructura-del-proyecto-y-capas)
4. [Diagrama de Flujo General del Proyecto](#4-diagrama-de-flujo-general-del-proyecto)
5. [Patrón Strategy — Proveedores de IA](#5-patrón-strategy--proveedores-de-ia)
6. [Patrón Factory — Creación de Scrapers](#6-patrón-factory--creación-de-scrapers)
7. [Patrón Template Method — Algoritmo de Scraping](#7-patrón-template-method--algoritmo-de-scraping)
8. [Cómo se conecta todo: Inyección de Dependencias](#8-cómo-se-conecta-todo-inyección-de-dependencias)
9. [Beneficios obtenidos y conclusiones](#9-beneficios-obtenidos-y-conclusiones)

---

## 1. Introducción al Proyecto

El proyecto es un **web scraper** que extrae productos de tres tiendas de e-commerce colombianas — **Éxito**, **Jumbo** y **Falabella** — y luego analiza esos datos con **inteligencia artificial local** utilizando modelos como Llama 3.2 (vía Ollama) o Gemma (vía LM Studio).

El sistema funciona en dos fases:

- **Fase 1 — Scraping**: Navega a las tiendas usando Selenium con Chrome en modo headless, extrae nombre, precio y URL de cada producto, y guarda los resultados en un archivo `results.json`.
- **Fase 2 — Análisis IA**: Toma los productos extraídos, los envía a un modelo de lenguaje local y genera un análisis comparativo en `ai_summary.md`.

El proyecto se ejecuta desde la línea de comandos y acepta parámetros para configurar qué tiendas consultar, cuántas páginas recorrer, qué proveedor de IA usar, entre otros. En la última ejecución obtuve **99 productos**: 16 de Éxito, 27 de Jumbo y 56 de Falabella.

---

## 2. ¿Por qué Clean Architecture?

### ¿Qué es Clean Architecture?

Clean Architecture es un enfoque de diseño propuesto por Robert C. Martin ("Uncle Bob") que organiza el código en **capas concéntricas**, donde la regla fundamental es que **las dependencias siempre apuntan hacia adentro**: las capas externas conocen a las internas, pero **nunca al revés**.

```
┌───────────────────────────────────────────────────┐
│              Presentation (CLI)                    │  ← Capa más externa
├───────────────────────────────────────────────────┤
│         Application (Casos de Uso)                 │
├───────────────────────────────────────────────────┤
│      Infrastructure (Scrapers, APIs, Storage)      │
├───────────────────────────────────────────────────┤
│    Domain (Entidades, Interfaces, Value Objects)   │  ← Núcleo (no depende de nada)
└───────────────────────────────────────────────────┘
```

### ¿Por qué la elegí sobre otras arquitecturas?

Evaluando las alternativas, Clean Architecture fue la mejor opción para este proyecto por estas razones:

| Arquitectura alternativa | Por qué no la elegí |
|:-------------------------|:---------------------|
| **MVC (Model-View-Controller)** | MVC es ideal para aplicaciones web con UI, pero mi proyecto es un CLI sin interfaz gráfica. Además, MVC no define reglas claras de dependencia entre capas. |
| **Arquitectura monolítica simple** | Con un solo archivo o pocos módulos habría sido rápido, pero no permite demostrar patrones de diseño ni escalabilidad. Si mañana necesito agregar una cuarta tienda, tendría que modificar código existente. |
| **Hexagonal (Ports & Adapters)** | Es muy similar a Clean Architecture y también sería válida. Elegí Clean Architecture porque su estructura de carpetas es más intuitiva para un proyecto con múltiples capas claramente diferenciadas. |
| **Microservicios** | Sería over-engineering para un proyecto de esta escala. Clean Architecture me da la separación que necesito sin la complejidad operativa de múltiples servicios. |

### Beneficios que obtuve

1. **Independencia del framework**: Mi lógica de negocio no depende de Selenium ni de Ollama. Si mañana cambio Selenium por Playwright, solo toco la capa de Infrastructure.
2. **Testeable**: Puedo testear los casos de uso sin necesidad de abrir un navegador ni tener Ollama corriendo — actualmente tengo **43 tests unitarios pasando**.
3. **Extensible**: Para agregar una nueva tienda solo creo una clase nueva y la registro en la fábrica. No modifico nada existente (principio Open/Closed).
4. **Separación de responsabilidades**: Cada capa tiene una responsabilidad clara. El Domain define el "qué", la Infrastructure define el "cómo", y la Application orquesta el "cuándo".

---

## 3. Estructura del Proyecto y Capas

### Estructura de carpetas

```
src/
├── main.py                                    ← Punto de entrada
│
├── Domain/                                    ← CAPA 1: NÚCLEO
│   ├── Entities/
│   │   ├── product.py                         → Product (dataclass)
│   │   └── ai_response.py                     → AIResponse (dataclass)
│   ├── Interfaces/
│   │   ├── scraper_interface.py               → IProductScraper, IScraperFactory
│   │   ├── llm_interface.py                   → ILLMStrategy, ILLMContext
│   │   └── repository_interface.py            → IProductRepository, IAIResponseRepository
│   └── ValueObjects/
│       └── scraping_config.py                 → StoreType, LLMProvider, ScrapingConfig, AIConfig
│
├── Application/                               ← CAPA 2: CASOS DE USO
│   ├── DTOs/
│   │   └── result_dtos.py                     → ScrapingResultDTO, AIAnalysisResultDTO, PipelineResultDTO
│   ├── Services/
│   │   └── application_service.py             → ApplicationService (fachada)
│   └── UseCases/
│       ├── scraping_use_case.py               → ScrapingUseCase
│       ├── ai_analysis_use_case.py            → AIAnalysisUseCase
│       └── pipeline_use_case.py               → PipelineUseCase
│
├── Infrastructure/                            ← CAPA 3: IMPLEMENTACIONES
│   ├── Repositories/
│   │   ├── base_scraper.py                    → BaseScraper (Template Method)
│   │   ├── exito_scraper.py                   → ExitoScraper
│   │   ├── jumbo_scraper.py                   → JumboScraper
│   │   ├── falabella_scraper.py               → FalabellaScraper
│   │   ├── scraper_factory.py                 → ScraperFactory (Factory)
│   │   ├── product_repository.py              → ProductRepository
│   │   └── ai_response_repository.py          → AIResponseRepository
│   └── ExternalServices/
│       ├── llm_base.py                        → BaseLLMStrategy
│       ├── ollama_strategy.py                 → OllamaStrategy
│       ├── lm_studio_strategy.py              → LMStudioStrategy
│       └── llm_context.py                     → LLMContext (Strategy)
│
└── Presentation/                              ← CAPA 4: INTERFAZ
    └── Controllers/
        └── cli_controller.py                  → run_cli(), parseo de argumentos
```

### Explicación de cada capa

#### 🔵 Domain — El corazón del sistema

Es la capa más interna y la más importante. **No depende de absolutamente nada externo** — ni de Selenium, ni de requests, ni de ninguna librería de terceros. Solo usa módulos estándar de Python (`dataclasses`, `abc`, `enum`, `typing`).

Aquí viven:

- **Entities** (`Product`, `AIResponse`): Son las estructuras de datos centrales. Un `Product` tiene nombre, precio, tienda, categoría, URL y fecha de extracción. Un `AIResponse` tiene la consulta, la respuesta del modelo y el proveedor que la generó.
- **Interfaces** (ABCs): Son los **contratos** que definen qué operaciones existen, sin decir cómo se implementan. Por ejemplo, `IProductScraper` dice "debe existir un método `scrape_category()`" pero no dice si usa Selenium, Playwright o una API REST.
- **Value Objects** (`StoreType`, `LLMProvider`, `ScrapingConfig`, `AIConfig`): Son objetos inmutables que representan configuraciones y tipos. `StoreType` es un enum con `EXITO`, `JUMBO`, `FALABELLA`.

El código de `Product` se ve así:

```python
@dataclass
class Product:
    name: str
    price: float
    store: str
    category: str
    rating: Optional[float] = None
    url: Optional[str] = None
    image_url: Optional[str] = None
    extracted_at: datetime = field(default_factory=datetime.now)
```

Es limpio, simple y no sabe nada sobre cómo se extrae ni cómo se almacena.

#### 🟢 Application — La lógica de orquestación

Esta capa contiene los **casos de uso** — es decir, las operaciones que el sistema puede realizar:

- `ScrapingUseCase`: Recibe una configuración, recorre las tiendas, usa la factory para crear el scraper correcto, ejecuta el scraping y guarda los productos.
- `AIAnalysisUseCase`: Carga productos guardados, crea el contexto de IA, ejecuta las consultas de análisis y guarda las respuestas.
- `PipelineUseCase`: Orquesta la ejecución completa — primero ejecuta scraping y luego análisis de IA.
- `ApplicationService`: Es una **fachada** que simplifica todo. El Controller solo necesita hablar con esta clase para ejecutar cualquier operación.

#### 🟠 Infrastructure — Las implementaciones concretas

Aquí es donde vive la "maquinaria real" — todo lo que toca el mundo exterior:

- **Scrapers**: `BaseScraper` (con Selenium), `ExitoScraper`, `JumboScraper`, `FalabellaScraper` — navegan las tiendas reales.
- **Servicios de IA**: `OllamaStrategy` (llama a `localhost:11434`), `LMStudioStrategy` (llama a `localhost:1234`).
- **Repositorios**: `ProductRepository` (guarda/carga JSON), `AIResponseRepository` (guarda Markdown).

Cada clase de Infrastructure **implementa una interfaz del Domain**. Esto es clave: el código de Application trabaja con las interfaces (el contrato), no con las implementaciones concretas.

#### 🔴 Presentation — La interfaz con el usuario

La capa más externa. En este proyecto es un CLI (Command Line Interface):

- `cli_controller.py` parsea los argumentos del usuario (`--stores`, `--category`, `--pages`, `--provider`, `--model`, `--skip-ai`, etc.).
- Crea un `ApplicationService` y llama a `run_full_pipeline()`.
- `main.py` solo hace: `from src.Presentation import run_cli` y ejecuta `run_cli()`.

---

## 4. Diagrama de Flujo General del Proyecto

> **Archivo**: `diagramas/01_flujo_general.drawio`

Este diagrama muestra el recorrido completo de una ejecución del sistema desde que el usuario escribe el comando hasta que se generan los archivos de salida.

### El flujo paso a paso

**1. Inicio y configuración**

El usuario ejecuta el programa desde la terminal:
```bash
python -m src.main --stores exito jumbo falabella --category tecnologia --pages 2 --provider ollama --model llama3.2
```

El `cli_controller.py` parsea estos argumentos y crea las configuraciones `ScrapingConfig` y `AIConfig`. Luego instancia un `ApplicationService` que a su vez crea todas las dependencias necesarias (scrapers, repositorios, casos de uso).

**2. PipelineUseCase toma el control**

`ApplicationService` llama a `PipelineUseCase.execute()`, que es el orquestador principal. Este caso de uso divide la ejecución en dos fases.

**3. Fase 1 — Scraping** (`ScrapingUseCase`)

Para cada tienda configurada:
1. `ScraperFactory` crea el scraper correcto (por ejemplo, `ExitoScraper` para Éxito).
2. El scraper ejecuta `scrape_category()` — el Template Method que abre Chrome, navega a la URL, hace scroll, extrae productos con BeautifulSoup y cierra el navegador.
3. Los productos se acumulan y se guardan en `output/results.json` vía `ProductRepository`.

**4. Decisión: ¿skip-ai?**

Si el usuario pasó `--skip-ai`, el pipeline termina aquí. Si no, continúa a la Fase 2.

**5. Fase 2 — Análisis IA** (`AIAnalysisUseCase`)

1. Se crea un `LLMContext` con la estrategia correspondiente (Ollama o LM Studio).
2. Se verifica que el proveedor esté disponible (`is_available()`).
3. Se cargan los productos desde `results.json`.
4. Se ejecutan las consultas de análisis — cada una genera una llamada al modelo de IA con reintentos automáticos.
5. Las respuestas se guardan en `output/ai_summary.md` vía `AIResponseRepository`.

**6. Fin**

Se muestra un resumen en consola: cantidad de productos extraídos, tiempo total, archivos generados.

---

## 5. Patrón Strategy — Proveedores de IA

> **Archivo**: `diagramas/02_patron_strategy.drawio`

### ¿Qué es el patrón Strategy?

Es un patrón de diseño de comportamiento que permite definir una **familia de algoritmos**, encapsular cada uno en una clase separada, y hacerlos **intercambiables** en tiempo de ejecución. El cliente (quien usa el algoritmo) no necesita saber cuál implementación concreta está usando — solo trabaja con la interfaz.

Es como tener un control remoto universal: el botón "encender" siempre hace lo mismo desde tu perspectiva, pero internamente puede estar hablando con un TV Samsung, un LG o un Sony.

### ¿Por qué lo usé aquí?

Necesitaba soportar **dos proveedores de IA diferentes** — Ollama y LM Studio — que tienen APIs completamente distintas:

| Aspecto | Ollama | LM Studio |
|:--------|:-------|:----------|
| Endpoint | `POST /api/generate` | `POST /v1/chat/completions` |
| Puerto | `localhost:11434` | `localhost:1234` |
| Formato de payload | `{ model, prompt, stream }` | `{ model, messages: [{role, content}] }` |
| Verificación | `GET /api/tags` | `GET /v1/models` |

Sin Strategy, tendría que llenar el código de `if provider == "ollama"... elif provider == "lm_studio"...` en cada lugar donde se usa IA. Con Strategy, todo eso se encapsula.

### ¿Cómo está implementado?

**1. La interfaz** (Domain): `ILLMStrategy` define el contrato:

```python
class ILLMStrategy(ABC):
    @abstractmethod
    def is_available(self) -> bool: ...

    @abstractmethod
    def analyze_products(self, products_data: str, query: str) -> AIResponse: ...
```

**2. La clase base** (Infrastructure): `BaseLLMStrategy` implementa la lógica común — el método `_make_request()` con reintentos automáticos (2 reintentos con backoff de 5 y 10 segundos) y la construcción del prompt:

```python
def _make_request(self, endpoint: str, payload: dict, retries: int = 2) -> Optional[dict]:
    for attempt in range(retries + 1):
        try:
            response = requests.post(url, json=payload, timeout=300)
            response.raise_for_status()
            return response.json()
        except requests.exceptions.Timeout:
            time.sleep(10)  # backoff
```

**3. Las estrategias concretas**: Cada una sabe cómo hablar con su API:

- `OllamaStrategy` — construye el payload para `/api/generate` y extrae la respuesta del campo `response`.
- `LMStudioStrategy` — construye el payload con `messages` para `/v1/chat/completions` (formato OpenAI) y extrae de `choices[0].message.content`.

**4. El contexto** (`LLMContext`): Mantiene una referencia a `ILLMStrategy` y la inicializa según la configuración:

```python
class LLMContext(ILLMContext):
    def _initialize_strategy(self):
        if self._config.provider == LLMProvider.OLLAMA:
            self._strategy = OllamaStrategy(self._config)
        elif self._config.provider == LLMProvider.LM_STUDIO:
            self._strategy = LMStudioStrategy(self._config)
    
    def analyze(self, data, queries):
        for query in queries:
            response = self._strategy.analyze_products(data, query)
            results.append(response)
```

### Beneficio concreto

Si mañana quiero agregar soporte para **OpenAI GPT-4**, solo creo una clase `OpenAIStrategy` que implemente `ILLMStrategy`, y la registro en `LLMContext`. No toco ni una línea de los casos de uso ni del pipeline.

---

## 6. Patrón Factory — Creación de Scrapers

> **Archivo**: `diagramas/03_patron_factory.drawio`

### ¿Qué es el patrón Factory?

Es un patrón de diseño creacional que **centraliza la creación de objetos**. En lugar de que cada parte del código decida qué clase concreta instanciar (usando `if`/`elif` en múltiples lugares), existe un único punto — la fábrica — que recibe un identificador y retorna el objeto correcto.

Es como un restaurante: tú pides "hamburguesa" al mesero (la fábrica), y él sabe que tiene que ir a la cocina y preparar la hamburguesa específica. Tú no necesitas saber cómo se prepara ni entrar a la cocina.

### ¿Por qué lo usé aquí?

Tengo 3 tiendas, cada una con su propio scraper. Sin Factory, el código de `ScrapingUseCase` tendría:

```python
# ❌ SIN FACTORY — acoplamiento directo
if store == "exito":
    scraper = ExitoScraper()
elif store == "jumbo":
    scraper = JumboScraper()
elif store == "falabella":
    scraper = FalabellaScraper()
```

Esto viola el principio Open/Closed: cada nueva tienda requiere modificar este código. Con Factory:

```python
# ✅ CON FACTORY — desacoplado
scraper = self._factory.create_scraper(store_type)
```

### ¿Cómo está implementado?

**1. La interfaz** (Domain): `IScraperFactory`

```python
class IScraperFactory(ABC):
    @abstractmethod
    def create_scraper(self, store_type: str) -> IProductScraper: ...
    
    @abstractmethod
    def get_available_stores(self) -> List[str]: ...
```

**2. La implementación** (Infrastructure): `ScraperFactory`

```python
class ScraperFactory(IScraperFactory):
    _scrapers: Dict[StoreType, Type[BaseScraper]] = {
        StoreType.EXITO: ExitoScraper,
        StoreType.JUMBO: JumboScraper,
        StoreType.FALABELLA: FalabellaScraper,
    }
    
    def create_scraper(self, store_type: str) -> IProductScraper:
        store = StoreType.from_string(store_type)
        scraper_class = self._scrapers.get(store)
        if not scraper_class:
            raise ValueError(f"Tienda no soportada: {store_type}")
        return scraper_class()
```

El diccionario `_scrapers` es el **registro central**: mapea cada `StoreType` (un enum) a la clase concreta del scraper. El método `create_scraper()` busca en ese diccionario, instancia la clase y la retorna como `IProductScraper`.

**3. El método de extensibilidad**: `register_scraper()`

```python
@classmethod
def register_scraper(cls, store_type: StoreType, scraper_class: Type[BaseScraper]):
    cls._scrapers[store_type] = scraper_class
```

Este método permite registrar nuevos scrapers **sin modificar la clase existente**. Si mañana agrego un scraper para Mercado Libre, solo hago:

```python
ScraperFactory.register_scraper(StoreType.MERCADOLIBRE, MercadoLibreScraper)
```

**4. El flujo de uso** en `ScrapingUseCase`:

1. El caso de uso recibe la lista de tiendas a consultar.
2. Para cada tienda, llama a `factory.create_scraper("exito")`.
3. La fábrica busca en su diccionario y retorna un `ExitoScraper`.
4. El caso de uso llama a `scraper.scrape_category()` — no sabe ni le importa que es un `ExitoScraper` internamente.

### Beneficio concreto

Para agregar Jumbo al proyecto (que fue exactamente lo que hice al reemplazar Alkosto), solo necesité: crear `JumboScraper`, agregarle la entrada al diccionario `_scrapers`, y agregar `JUMBO` al enum `StoreType`. **Cero cambios en los casos de uso, cero cambios en el pipeline, cero cambios en el CLI.** La fábrica se encarga de todo.

---

## 7. Patrón Template Method — Algoritmo de Scraping

> **Archivo**: `diagramas/04_patron_template_method.drawio`

### ¿Qué es el patrón Template Method?

Es un patrón de diseño de comportamiento que define el **esqueleto de un algoritmo** en una clase base, delegando algunos pasos específicos a las subclases. La clase base dice "primero haz A, luego B, luego C", y las subclases implementan B y C según su necesidad, pero **nunca cambian el orden** ni el flujo general.

Es como una receta de cocina: los pasos siempre son "preparar ingredientes → cocinar → servir", pero cada plato tiene su forma específica de preparar y cocinar.

### ¿Por qué lo usé aquí?

Los 3 scrapers (Éxito, Jumbo, Falabella) siguen **exactamente el mismo proceso general**:

1. Abrir un navegador Chrome en modo headless
2. Para cada página: construir la URL → navegar → hacer scroll → extraer productos
3. Cerrar el navegador
4. Retornar los productos

Lo que **cambia** entre tiendas es:
- **La URL**: Éxito usa `/tecnologia?page=2`, Jumbo usa `/tecnologia?page=2` con otra estructura, Falabella usa `/category/cat123?page=2`.
- **Los selectores CSS**: Éxito tiene 7 selectores fallback, Jumbo usa selectores VTEX, Falabella tiene los suyos.

Sin Template Method, tendría el mismo código de Selenium (abrir driver, scroll, cerrar) **copiado y pegado** en 3 archivos. Eso es exactamente lo que el patrón evita.

### ¿Cómo está implementado?

**1. La clase abstracta**: `BaseScraper` define el **método template** (`scrape_category`) y los **hooks**:

```python
class BaseScraper(IProductScraper):
    
    # TEMPLATE METHOD — define el algoritmo completo
    def scrape_category(self, category: str, num_pages: int = 2) -> List[Product]:
        all_products = []
        try:
            self._init_driver()                              # Paso 1: abrir Chrome
            for page in range(1, num_pages + 1):             # Paso 2: loop páginas
                url = self.get_category_url(category, page)  # 🔴 HOOK 1 (abstracto)
                html_content = self._fetch_page(url)         # Paso 3: navegar + scroll
                products = self.parse_products(html, cat)    # 🔴 HOOK 2 (abstracto)
                all_products.extend(products)
                self._wait_between_requests()                # Paso 4: esperar
        finally:
            self._close_driver()                             # Paso 5: cerrar Chrome
        return all_products
    
    # Hooks que las subclases DEBEN implementar
    @abstractmethod
    def get_category_url(self, category: str, page: int) -> str: ...
    
    @abstractmethod
    def parse_products(self, html_content: str, category: str) -> List[Product]: ...
    
    # Métodos concretos reutilizados por todos
    def _init_driver(self): ...      # Configura Chrome headless
    def _fetch_page(self, url): ...  # Selenium navega + scroll
    def _scroll_page(self): ...      # JavaScript scroll down
    def _close_driver(self): ...     # Cierra el navegador
    def _clean_price(self, text): ...# Parsea "$1.234.567" → 1234567.0
```

**2. Las subclases** implementan solo los hooks:

**ExitoScraper** — 7 selectores CSS de fallback para encontrar productos:
```python
class ExitoScraper(BaseScraper):
    def __init__(self):
        self._store_name = "Exito"
        self._base_url = "https://www.exito.com"
    
    def get_category_url(self, category, page):
        return f"{self._base_url}/{category}?page={page}"
    
    def parse_products(self, html_content, category):
        # Usa BeautifulSoup con 7 selectores CSS fallback
        # Extrae nombre, precio, URL de cada producto
```

**JumboScraper** — selectores específicos de la plataforma VTEX:
```python
class JumboScraper(BaseScraper):
    def get_category_url(self, category, page):
        return f"https://www.tiendasjumbo.co/{category}?page={page}"
    
    def parse_products(self, html_content, category):
        # Selectores VTEX: .vtex-product-summary, .vtex-store-components
```

**FalabellaScraper** — selectores de la estructura de Falabella Colombia:
```python
class FalabellaScraper(BaseScraper):
    def get_category_url(self, category, page):
        return f"https://www.falabella.com.co/falabella-co/category/{category}?page={page}"
    
    def parse_products(self, html_content, category):
        # Selectores propios de Falabella
```

### El flujo visual

En el diagrama se ve claramente:

1. **Azul** = métodos concretos de `BaseScraper` (se reutilizan tal cual)
2. **Rojo** = **HOOKS** (cada subclase los implementa diferente)
3. La flecha de "siguiente página" vuelve al inicio del loop

```
scrape_category()
    │
    ├── _init_driver()              ← [BaseScraper] Chrome headless
    │
    ├── 🔄 Para cada página:
    │   ├── get_category_url()     ← [HOOK] URL específica por tienda
    │   ├── _fetch_page()          ← [BaseScraper] Selenium navega
    │   ├── _scroll_page()         ← [BaseScraper] JS scroll
    │   ├── parse_products()       ← [HOOK] Selectores CSS específicos
    │   └── _wait()                ← [BaseScraper] Anti-bot delay
    │
    └── _close_driver()            ← [BaseScraper] Cierra Chrome
```

### Beneficio concreto

Si Éxito cambia su HTML mañana, solo modifico `ExitoScraper.parse_products()`. No toco el flujo de Selenium, ni el scroll, ni el manejo de errores, ni el cierre del driver — todo eso se hereda de `BaseScraper`.

---

## 8. Cómo se conecta todo: Inyección de Dependencias

El `ApplicationService` actúa como el **Composition Root** — es el único lugar donde se crean las instancias concretas y se inyectan:

```python
class ApplicationService:
    def __init__(self, output_dir="output"):
        # Crear implementaciones concretas
        self._product_repo = ProductRepository(output_dir)
        self._ai_repo = AIResponseRepository(output_dir)
        self._scraper_factory = ScraperFactory()
        
        # Inyectarlas en los casos de uso
        self._scraping_use_case = ScrapingUseCase(
            self._scraper_factory,    # ← inyecta la fábrica
            self._product_repo        # ← inyecta el repositorio
        )
        self._ai_use_case = AIAnalysisUseCase(
            self._product_repo,       # ← comparte el mismo repositorio
            self._ai_repo
        )
        self._pipeline_use_case = PipelineUseCase(
            self._scraping_use_case,  # ← inyecta los casos de uso
            self._ai_use_case
        )
```

Los **casos de uso nunca crean sus propias dependencias** — las reciben por constructor. Esto es **inyección de dependencias manual** (sin framework). Es simple, explícita y funciona perfectamente para un proyecto de esta escala.

---

## 9. Beneficios obtenidos y conclusiones

### Resumen de patrones y dónde actúan

| Patrón | Dónde | Problema que resuelve |
|:-------|:------|:---------------------|
| **Strategy** | Proveedores de IA | Intercambiar Ollama ↔ LM Studio sin cambiar código cliente |
| **Factory** | Creación de scrapers | Centralizar la creación, permitir extensibilidad con `register_scraper()` |
| **Template Method** | Algoritmo de scraping | Reutilizar el flujo de Selenium, personalizar solo URL y parseo |

### Beneficios medidos en la práctica

- **43 tests unitarios** corriendo sin necesidad de infraestructura externa (ni Chrome ni Ollama).
- **3 tiendas** funcionando con **99 productos** en la última ejecución.
- **Reemplazo de Alkosto por Jumbo** en menos de 30 minutos: crear el scraper + registrar en la factory + actualizar el enum. **Cero cambios** en Application ni Presentation.
- **2 proveedores de IA** intercambiables con un solo parámetro `--provider`.
- **Reintentos automáticos** con backoff en la capa de Strategy, transparentes para los casos de uso.

### Conclusión

Clean Architecture me dio la estructura para que cada parte del sistema tenga su responsabilidad clara. Los patrones de diseño resolven problemas concretos: Strategy me permite cambiar de proveedor de IA sin tocar lógica de negocio, Factory centraliza la creación de scrapers haciéndolo extensible, y Template Method elimina la duplicación del flujo de Selenium entre las 3 tiendas.

El resultado es un proyecto que es **fácil de entender** (cada archivo tiene una responsabilidad), **fácil de extender** (agregar una tienda o un proveedor no requiere modificar código existente), y **fácil de testear** (las interfaces permiten crear mocks para los tests).
