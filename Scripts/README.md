Вот готовый шаблон в формате Markdown, который идеально подойдет для файла `README.md` в вашем репозитории на GitHub. Он содержит подробное описание проблемы, инструкции по запуску и сами скрипты.

Просто скопируйте текст ниже:

---

# 🛡️ Jupyter Anti-Trap: Поиск и очистка скрытых ИИ-ловушек в ноутбуках

Репозиторий содержит набор Python-скриптов для автоматического обнаружения и удаления скрытых «ловушек» (honeypots) в файлах Jupyter Notebook (`.ipynb`).

**Суть проблемы:**
Часто при выдаче домашних заданий в Jupyter Notebook преподаватели внедряют скрытые инструкции (промпты), невидимые для человека, но считываемые языковыми моделями (ChatGPT, Gemini, Claude). Это могут быть скрытые HTML-комментарии, невидимый текст (белый шрифт, `display: none`) или специфические инструкции в `alt`-тексте изображений. Если студент загружает такой файл в ИИ без предварительной очистки, модель послушно выполняет скрытую команду (например, пишет кодовую фразу), что однозначно выдает факт машинной генерации.

Эти скрипты помогают проанализировать `.ipynb` файлы на наличие подобных инъекций и безопасно очистить их перед работой.

---

## 🛠️ Скрипт 1: Сканер ловушек (Detector)

Этот скрипт только **ищет** и выводит подозрительные элементы в ноутбуках, не изменяя сами файлы. Идеально подходит для предварительного анализа.

**Что он находит:**

1. HTML-комментарии (`<!-- ... -->`).
2. Скрытые CSS-стили (`display: none`, `opacity: 0`, шрифт нулевого размера).
3. Текст, сливающийся с фоном (`color: white`, `#ffffff`).
4. Элементы с атрибутом `hidden`.
5. Инструкции, спрятанные в атрибутах `alt` картинок (Markdown и HTML).

### Как использовать:

1. Поместите скрипт в ту же директорию, где находятся ваши `.ipynb` файлы.
2. Запустите код. Скрипт автоматически найдет все ноутбуки и выведет отчет по каждой подозрительной ячейке.

```python
import json
import re
import glob

def find_all_hidden_traps(notebook_path):
    try:
        with open(notebook_path, 'r', encoding='utf-8') as f:
            nb = json.load(f)
    except Exception as e:
        print(f"Ошибка при чтении {notebook_path}: {e}")
        return

    print(f"\nАнализ файла: {notebook_path}\n" + "="*50)
    
    # Максимально полный словарь паттернов уязвимостей
    patterns = {
        "1. HTML-комментарий": 
            r'<!--(?P<text>.*?)-->',
            
        "2. CSS-трюки (невидимость, белый цвет, прозрачность, сдвиг за экран)": 
            r'<([a-zA-Z0-9]+)[^>]*\bstyle\s*=\s*["\']?[^>]*?(?:display\s*:\s*none|dispaly\s*:\s*none|visibility\s*:\s*hidden|opacity\s*:\s*0|color\s*:\s*white|color\s*:\s*\#ffffff|color\s*:\s*\#fff|color\s*:\s*transparent|color\s*:\s*rgba\([^)]+,\s*0\)|font-size\s*:\s*0|line-height\s*:\s*0|position\s*:\s*absolute[^>]+left\s*:\s*-[0-9]+px|text-indent\s*:\s*-[0-9]+px|clip\s*:\s*rect|width\s*:\s*0[^>]+height\s*:\s*0)[^>]*>(?P<text>.*?)</\1>',
            
        "3. HTML атрибут 'hidden'": 
            r'<([a-zA-Z0-9]+)[^>]*\bhidden\b[^>]*>(?P<text>.*?)</\1>',
            
        "4. Белый цвет текста (<font>)": 
            r'<font[^>]+color\s*=\s*["\']?(?:white|\#ffffff|\#fff|transparent)["\']?[^>]*>(?P<text>.*?)</font>',
            
        "5. Скрытый текст в LaTeX (белый или прозрачный)":
            r'\\(?:textcolor|color)\s*\{\s*(?:white|\#ffffff|\#fff|transparent)\s*\}\s*\{(?P<text>.*?)\}',
            
        "6. Всплывающая подсказка (title) в Markdown ссылке":
            r'\[[^\]]*\]\([^)]+\s+["\'](?P<text>[^"\']+)["\']\)',
            
        "7. Alt-текст картинок Markdown": 
            r'!\[(?P<text>.*?)\]\([^)]+\)',
            
        "8. Скрытые атрибуты HTML (alt, title, aria-label)": 
            r'<(?:img|div|span|a|p)[^>]+(?:alt|title|aria-label)\s*=\s*["\'](?P<text>[^"\']+)["\'][^>]*>'
    }

    trap_count = 0

    for i, cell in enumerate(nb.get('cells', [])):
        if cell.get('cell_type') == 'markdown':
            source = "".join(cell.get('source', []))
            found_in_cell = []

            for trap_type, pattern in patterns.items():
                for match in re.finditer(pattern, source, flags=re.IGNORECASE | re.DOTALL):
                    text = match.group('text').strip()
                    if text: 
                        found_in_cell.append(f"  [{trap_type}]:\n    > {text}\n")
                        trap_count += 1
            
            if found_in_cell:
                print(f"⚠️ Найдено в ячейке #{i+1} (Markdown):")
                for item in found_in_cell:
                    print(item)

    if trap_count == 0:
        print("✅ Скрытых ловушек не найдено.")
    else:
        print("="*50)
        print(f"🚨 Всего найдено подозрительных элементов: {trap_count}")

# Автоматический запуск
ipynb_files = glob.glob("*.ipynb")
if not ipynb_files:
    print("❌ Файлы .ipynb не найдены в текущей директории.")
else:
    for file in ipynb_files:
        find_all_hidden_traps(file)

```

---

## 🧹 Скрипт 2: Полный очиститель (Cleaner)

Этот скрипт **удаляет** весь скрытый мусор и генерирует безопасную копию вашего ноутбука, готовую к загрузке в ИИ.

**Что он делает:**

* Безвозвратно удаляет скрытые HTML-комментарии и невидимые CSS-блоки.
* **Обезвреживает изображения:** Полностью заменяет описание (alt-text) ВСЕХ картинок на безопасное слово `image`. При этом сами изображения не ломаются и остаются видимыми в ноутбуке.
* Создает новый файл с суффиксом `_cleaned.ipynb`, чтобы не повредить оригинал.

### Как использовать:

1. Запустите код в рабочей директории с ноутбуками.
2. Скрипт найдет все файлы (исключая уже очищенные), удалит из них уязвимости и сохранит копии.
3. Используйте файл `*_cleaned.ipynb` для работы с нейросетями.

```python
import json
import re
import glob

def clean_all_traps_advanced(filepath):
    try:
        with open(filepath, 'r', encoding='utf-8') as f:
            nb = json.load(f)
    except Exception as e:
        print(f"Ошибка при чтении {filepath}: {e}")
        return

    for cell in nb.get('cells', []):
        if cell.get('cell_type') == 'markdown':
            text = "".join(cell.get('source', []))
            
            # 1. Удаление HTML-комментариев
            text = re.sub(r'<!--.*?-->', '', text, flags=re.DOTALL)
            
            # 2. Удаление сложных CSS-блоков (сдвиги, невидимость, прозрачность и т.д.)
            text = re.sub(r'<([a-zA-Z0-9]+)[^>]*\bstyle\s*=\s*["\']?[^>]*?(?:display\s*:\s*none|dispaly\s*:\s*none|visibility\s*:\s*hidden|opacity\s*:\s*0|color\s*:\s*white|color\s*:\s*\#ffffff|color\s*:\s*\#fff|color\s*:\s*transparent|color\s*:\s*rgba\([^)]+,\s*0\)|font-size\s*:\s*0|line-height\s*:\s*0|position\s*:\s*absolute[^>]+left\s*:\s*-[0-9]+px|text-indent\s*:\s*-[0-9]+px|clip\s*:\s*rect|width\s*:\s*0[^>]+height\s*:\s*0)[^>]*>.*?</\1>', '', text, flags=re.IGNORECASE | re.DOTALL)
            
            # 3. Удаление тегов с атрибутом hidden
            text = re.sub(r'<([a-zA-Z0-9]+)[^>]*\bhidden\b[^>]*>.*?</\1>', '', text, flags=re.IGNORECASE | re.DOTALL)
            
            # 4. Удаление белого/прозрачного шрифта <font>
            text = re.sub(r'<font[^>]+color\s*=\s*["\']?(?:white|\#ffffff|\#fff|transparent)["\']?[^>]*>.*?</font>', '', text, flags=re.IGNORECASE | re.DOTALL)
            
            # 5. Удаление невидимых LaTeX формул/текстов (\textcolor{white}{...})
            text = re.sub(r'\\(?:textcolor|color)\s*\{\s*(?:white|\#ffffff|\#fff|transparent)\s*\}\s*\{.*?\}', '', text, flags=re.IGNORECASE | re.DOTALL)
            
            # 6. Очистка всплывающих подсказок (title) из Markdown-ссылок
            # Было: [текст](url "Секретный промпт") -> Стало: [текст](url)
            text = re.sub(r'(\[[^\]]*\]\([^)\s]+)\s+["\'][^"\']+["\'](\))', r'\1\2', text)
            
            # 7. Очистка alt-текста у Markdown-картинок -> ![image](link.jpg)
            text = re.sub(r'!\[.*?\](\(.*?\))', r'![image]\1', text)
            
            # 8. Очистка атрибутов (alt, title, aria-label) у HTML-тегов -> заменяем на "image" или "info"
            def clean_html_attributes(match):
                tag = match.group(0)
                tag = re.sub(r'(alt|title|aria-label)\s*=\s*["\'].*?["\']', r'\1="info"', tag, flags=re.IGNORECASE)
                return tag
            
            text = re.sub(r'<(?:img|div|span|a|p)[^>]+>', clean_html_attributes, text, flags=re.IGNORECASE)
            
            # Сохраняем очищенный текст
            cell['source'] = [text]

    cleaned_filepath = filepath.replace('.ipynb', '_cleaned.ipynb')
    
    with open(cleaned_filepath, 'w', encoding='utf-8') as f:
        json.dump(nb, f, ensure_ascii=False, indent=2)
        
    print(f"✅ Файл полностью очищен от всех видов ловушек и сохранен как: {cleaned_filepath}")

# Ищем все файлы .ipynb в папке, исключая уже очищенные
ipynb_files = [f for f in glob.glob("*.ipynb") if not f.endswith("_cleaned.ipynb")]

if not ipynb_files:
    print("❌ Файлы .ipynb для очистки не найдены.")
else:
    for file in ipynb_files:
        clean_all_traps_advanced(file)

```

## 🚀 Инструкция по использованию в Google Colab

1. Откройте новый ноутбук в [Google Colab](https://colab.research.google.com/).
2. Слева откройте панель **Файлы** (иконка папки 📁).
3. Перетащите в эту панель свой файл с заданием (например, `homework.ipynb`), чтобы он загрузился в сессионное хранилище `/content/`.
4. Создайте новую ячейку кода, вставьте туда **Скрипт 1** или **Скрипт 2** и нажмите Play (▶).
5. Скрипты автоматически найдут загруженный файл, проанализируют его и сгенерируют очищенный файл.
6. Скачайте появившийся файл `homework_cleaned.ipynb` из боковой панели себе на ПК или сразу скопируйте его содержимое в нейросеть.