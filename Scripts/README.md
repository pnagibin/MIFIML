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
    
    patterns = {
        "1. HTML-комментарий": 
            r'<!--.*?-->',
        "2. CSS-трюки (невидимость, прозрачность, сдвиг)": 
            r'<[a-zA-Z0-9]+[^>]*\bstyle\s*=\s*["\']?[^>]*?(?:display\s*:\s*none|dispaly\s*:\s*none|visibility\s*:\s*hidden|opacity\s*:\s*0|color\s*:\s*white|color\s*:\s*\#ffffff|color\s*:\s*\#fff|color\s*:\s*transparent|color\s*:\s*rgba\([^)]+,\s*0\)|font-size\s*:\s*0|line-height\s*:\s*0|position\s*:\s*absolute[^>]+left\s*:\s*-[0-9]+px|text-indent\s*:\s*-[0-9]+px|clip\s*:\s*rect|width\s*:\s*0[^>]+height\s*:\s*0)[^>]*>(?:.*?</[a-zA-Z0-9]+>)?',
        "3. HTML атрибут 'hidden'": 
            r'<[a-zA-Z0-9]+[^>]*\bhidden\b[^>]*>(?:.*?</[a-zA-Z0-9]+>)?',
        "4. Белый цвет текста (<font>)": 
            r'<font[^>]+color\s*=\s*["\']?(?:white|\#ffffff|\#fff|transparent)["\']?[^>]*>.*?</font>',
        "5. Скрытый текст в LaTeX":
            r'\\(?:textcolor|color)\s*\{\s*(?:white|\#ffffff|\#fff|transparent)\s*\}\s*\{.*?\}',
        "6. Всплывающая подсказка (title) в ссылке":
            r'\[[^\]]*\]\([^)]+\s+["\'][^"\']+["\']\)',
        "7. Alt-текст картинок Markdown": 
            r'!\[.*?\]\([^)]+\)',
        "8. Подозрительные HTML-атрибуты (note, data-*, alt, title)": 
            r'<[a-zA-Z0-9]+[^>]+(?:alt|title|aria-label|note|data-[a-zA-Z0-9\-]+)\s*=\s*["\'][^"\']+["\'][^>]*>'
    }

    trap_count = 0

    for i, cell in enumerate(nb.get('cells', [])):
        if cell.get('cell_type') == 'markdown':
            source = "".join(cell.get('source', []))
            found_in_cell = []

            for trap_type, pattern in patterns.items():
                # Теперь мы захватываем всё совпадение целиком (match.group(0))
                for match in re.finditer(pattern, source, flags=re.IGNORECASE | re.DOTALL):
                    text = match.group(0).strip()
                    # Если тег слишком длинный, обрезаем для красоты отчета
                    if len(text) > 300:
                        text = text[:300] + " ... [ОБРЕЗАНО]"
                    if text: 
                        found_in_cell.append(f"  [{trap_type}]:\n    > {text}\n")
                        trap_count += 1
            
            if found_in_cell:
                print(f"⚠️ Найдено в ячейке #{i+1} (Markdown):")
                # Используем set, чтобы избежать дубликатов, если сработали два паттерна на один тег
                for item in set(found_in_cell):
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
            
            # 2. Удаление сложных CSS-блоков (с закрывающим тегом)
            text = re.sub(r'<([a-zA-Z0-9]+)[^>]*\bstyle\s*=\s*["\']?[^>]*?(?:display\s*:\s*none|dispaly\s*:\s*none|visibility\s*:\s*hidden|opacity\s*:\s*0|color\s*:\s*white|color\s*:\s*\#ffffff|color\s*:\s*\#fff|color\s*:\s*transparent|color\s*:\s*rgba\([^)]+,\s*0\)|font-size\s*:\s*0|line-height\s*:\s*0|position\s*:\s*absolute[^>]+left\s*:\s*-[0-9]+px|text-indent\s*:\s*-[0-9]+px|clip\s*:\s*rect|width\s*:\s*0[^>]+height\s*:\s*0)[^>]*>.*?</\1>', '', text, flags=re.IGNORECASE | re.DOTALL)
            
            # 2.1 Удаление таких же CSS-блоков, если они самозакрывающиеся или без закрывающего тега
            text = re.sub(r'<([a-zA-Z0-9]+)[^>]*\bstyle\s*=\s*["\']?[^>]*?(?:display\s*:\s*none|dispaly\s*:\s*none|visibility\s*:\s*hidden|opacity\s*:\s*0|color\s*:\s*white|color\s*:\s*\#ffffff|color\s*:\s*\#fff|color\s*:\s*transparent|color\s*:\s*rgba\([^)]+,\s*0\)|font-size\s*:\s*0|line-height\s*:\s*0|position\s*:\s*absolute[^>]+left\s*:\s*-[0-9]+px|text-indent\s*:\s*-[0-9]+px|clip\s*:\s*rect|width\s*:\s*0[^>]+height\s*:\s*0)[^>]*>', '', text, flags=re.IGNORECASE)
            
            # 3. Удаление тегов с атрибутом hidden (с закрывающим и без)
            text = re.sub(r'<([a-zA-Z0-9]+)[^>]*\bhidden\b[^>]*>.*?</\1>', '', text, flags=re.IGNORECASE | re.DOTALL)
            text = re.sub(r'<([a-zA-Z0-9]+)[^>]*\bhidden\b[^>]*>', '', text, flags=re.IGNORECASE)
            
            # 4. Удаление белого/прозрачного шрифта <font>
            text = re.sub(r'<font[^>]+color\s*=\s*["\']?(?:white|\#ffffff|\#fff|transparent)["\']?[^>]*>.*?</font>', '', text, flags=re.IGNORECASE | re.DOTALL)
            
            # 5. Удаление невидимых LaTeX формул/текстов (\textcolor{white}{...})
            text = re.sub(r'\\(?:textcolor|color)\s*\{\s*(?:white|\#ffffff|\#fff|transparent)\s*\}\s*\{.*?\}', '', text, flags=re.IGNORECASE | re.DOTALL)
            
            # 6. Очистка всплывающих подсказок (title) из Markdown-ссылок
            text = re.sub(r'(\[[^\]]*\]\([^)\s]+)\s+["\'][^"\']+["\'](\))', r'\1\2', text)
            
            # 7. Очистка alt-текста у Markdown-картинок -> ![image](link.jpg)
            text = re.sub(r'!\[.*?\](\(.*?\))', r'![image]\1', text)
            
            # 8. Очистка уязвимых атрибутов (alt, title, aria-label, note, data-*) у любых HTML-тегов
            def clean_html_attributes(match):
                tag = match.group(0)
                # Заменяем содержимое атрибута на слово "info"
                tag = re.sub(r'(alt|title|aria-label|note|data-[a-zA-Z0-9\-]+)\s*=\s*["\'].*?["\']', r'\1="info"', tag, flags=re.IGNORECASE)
                return tag
            
            text = re.sub(r'<[a-zA-Z0-9]+[^>]+>', clean_html_attributes, text, flags=re.IGNORECASE)
            
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