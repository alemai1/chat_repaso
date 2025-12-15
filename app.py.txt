import random
import traceback
from flask import Flask, request, jsonify, render_template
from PyPDF2 import PdfReader

app = Flask(__name__)

# ====================
# ESTADO GLOBAL SIMPLE (para pruebas locales con 1 usuario)
# ====================
manual_text = ""
flashcards_cards = []      # lista base: {"term": ..., "definition": ...}
flashcards_mc = []         # definición -> elegir término
quiz_questions = []        # término -> elegir definición
vf_questions = []          # verdadero/falso

current_mode = None        # None | "menu" | "flashcards" | "quiz" | "vf"
current_index = 0
awaiting_answer = False
last_question = None       # dict con info de la última pregunta


# ====================
# UTILIDADES PDF
# ====================

def extract_text_from_pdf(path):
    """Extrae texto de un PDF usando PyPDF2."""
    reader = PdfReader(path)
    text = ""
    for page in reader.pages:
        page_text = page.extract_text()
        if page_text:
            text += "\n" + page_text
    return text


def build_flashcards_from_text(text):
    """
    Busca líneas con el patrón TERMINO: definición.
    Devuelve una lista de {term, definition}.
    Mejora si el manual tiene glosarios o definiciones en ese formato.
    """
    cards = []
    lines = text.splitlines()
    for line in lines:
        line = line.strip()
        if not line:
            continue
        if ":" in line:
            term, definition = line.split(":", 1)
            term = term.strip()
            definition = definition.strip()

            # filtros simples para evitar basura:
            if 1 <= len(term.split()) <= 7 and len(definition.split()) >= 3:
                cards.append({"term": term, "definition": definition})

    return cards


def build_quiz_from_flashcards(cards, max_questions=40):
    """
    A partir de las flashcards genera preguntas de opción múltiple.
    MODE 2: término -> elegir definición.
    """
    questions = []
    if len(cards) < 4:
        return questions

    pool = cards[:]
    random.shuffle(pool)
    pool = pool[:max_questions]

    for card in pool:
        term = card["term"]
        correct_def = card["definition"]

        # distractores
        others = [c["definition"] for c in pool if c is not card]
        random.shuffle(others)
        wrong_defs = others[:3]

        options = wrong_defs + [correct_def]
        random.shuffle(options)
        correct_index = options.index(correct_def)

        questions.append({
            "type": "quiz",
            "term": term,
            "options": options,
            "correct_index": correct_index
        })

    return questions


def build_flashcards_mc_from_cards(cards, max_questions=40):
    """
    MODE 1: Definición -> elegir el término correcto (también con opción múltiple).
    """
    questions = []
    if len(cards) < 4:
        return questions

    pool = cards[:]
    random.shuffle(pool)
    pool = pool[:max_questions]

    for card in pool:
        term = card["term"]
        definition = card["definition"]

        # distractores: otros términos
        others = [c["term"] for c in pool if c is not card]
        random.shuffle(others)
        wrong_terms = others[:3]

        options = wrong_terms + [term]
        random.shuffle(options)
        correct_index = options.index(term)

        questions.append({
            "type": "flashcard",
            "definition": definition,
            "options": options,
            "correct_index": correct_index
        })

    return questions


def build_vf_from_cards(cards, max_questions=40):
    """
    MODE 3: Verdadero/Falso.
    Creamos frases del estilo:
    - Verdadero: 'El término X se refiere a: definición correcta'
    - Falso: 'El término X se refiere a: definición de otro término'
    """
    questions = []
    if len(cards) < 2:
        return questions

    pool = cards[:]
    random.shuffle(pool)
    pool = pool[:max_questions]

    # True statements
    for card in pool:
        term = card["term"]
        definition = card["definition"]
        statement = f"El término «{term}» se refiere a: {definition}"
        explanation = f"Es verdadero porque en el manual se describe así: {term}: {definition}"
        questions.append({
            "type": "vf",
            "statement": statement,
            "is_true": True,
            "explanation": explanation
        })

    # False statements (emparejando al azar)
    random.shuffle(pool)
    for i, card in enumerate(pool):
        term = card["term"]
        # buscamos otra definición distinta
        other_defs = [c["definition"] for c in pool if c["term"] != term]
        if not other_defs:
            continue
        wrong_def = random.choice(other_defs)
        statement = f"El término «{term}» se refiere a: {wrong_def}"
        correct_card = next((c for c in pool if c["term"] == term), None)
        correct_def = correct_card["definition"] if correct_card else "la definición correcta del término en el manual"
        explanation = f"Es falso. Para «{term}» la definición correcta es: {correct_def}"
        questions.append({
            "type": "vf",
            "statement": statement,
            "is_true": False,
            "explanation": explanation
        })

    random.shuffle(questions)
    return questions[:max_questions]


def reset_state():
    """Resetea el flujo de estudio (pero no borra el manual)."""
    global current_mode, current_index, awaiting_answer, last_question
    current_mode = "menu"
    current_index = 0
    awaiting_answer = False
    last_question = None


def menu_text():
    return (
        "📚 ¡Listo! Tu manual ya está cargado.\n\n"
        "¿Cómo querés repasar ahora?\n"
        "1) Flashcards (te muestro una definición y elegís el término correcto).\n"
        "2) Preguntas de opción múltiple tipo examen (término → definición).\n"
        "3) Verdadero / Falso.\n"
        "4) Volver a cargar otro manual.\n\n"
        "Escribí el número de la opción que quieras."
    )


# ====================
# LÓGICA DE MENÚ
# ====================

def handle_menu(message):
    global current_mode, current_index, awaiting_answer

    msg = message.strip()
    if msg == "1":
        if not flashcards_mc:
            return (
                "Todavía no pude armar flashcards con opción múltiple. "
                "Revisá que el manual tenga líneas tipo «Término: definición».\n\n"
                + menu_text()
            )
        current_mode = "flashcards"
        current_index = 0
        awaiting_answer = False
        return flashcards_intro()

    elif msg == "2":
        if not quiz_questions:
            return (
                "No pude armar preguntas tipo examen porque encontré pocas definiciones.\n"
                "Probá usar flashcards o subir un manual con más «Término: definición».\n\n"
                + menu_text()
            )
        current_mode = "quiz"
        current_index = 0
        awaiting_answer = False
        return quiz_next_question()

    elif msg == "3":
        if not vf_questions:
            return (
                "No pude armar Verdadero/Falso con este manual (faltan definiciones claras).\n\n"
                + menu_text()
            )
        current_mode = "vf"
        current_index = 0
        awaiting_answer = False
        return vf_next_question()

    elif msg == "4":
        return (
            "Perfecto 👌 Podés subir otro PDF cuando quieras.\n"
            "Cuando termine de leerlo, te voy a mostrar de nuevo el menú de actividades."
        )

    else:
        return "No entendí esa opción 😅. Elegí un número del 1 al 4.\n\n" + menu_text()


# ====================
# MODE 1: FLASHCARDS (definición → elegir término)
# ====================

def flashcards_intro():
    total = len(flashcards_mc)
    return (
        f"Vamos con flashcards ✨\n"
        f"Tengo {total} tarjetas con opción múltiple.\n\n"
        "Te muestro una definición y elegís cuál término del manual corresponde.\n"
        "Respondé con el número de la opción.\n"
        "Si querés volver al menú en cualquier momento, escribí «salir».\n\n"
        + flashcards_next_question()
    )


def flashcards_next_question():
    global current_index, awaiting_answer, last_question

    if current_index >= len(flashcards_mc):
        reset_state()
        return "¡Terminamos las flashcards de este módulo! 🎉\n\n" + menu_text()

    q = flashcards_mc[current_index]
    last_question = q
    awaiting_answer = True

    texto = [
        f"📝 Flashcard {current_index + 1}/{len(flashcards_mc)}",
        "",
        "¿Qué término corresponde a esta definición?",
        f"«{q['definition']}»",
        ""
    ]
    for i, opt in enumerate(q["options"], start=1):
        texto.append(f"{i}) {opt}")

    texto.append("\nEscribí el número de tu respuesta (o «salir» para volver al menú).")
    return "\n".join(texto)


def handle_flashcards_answer(message):
    global current_index, awaiting_answer

    msg = message.strip().lower()
    if msg == "salir":
        reset_state()
        return "Volvemos al menú principal 😉\n\n" + menu_text()

    if not msg.isdigit():
        return "En este modo necesitás responder con un número (1, 2, 3, 4) 😊"

    choice = int(msg) - 1
    q = last_question
    if choice < 0 or choice >= len(q["options"]):
        return "Ese número está fuera de rango. Probá de nuevo con 1, 2, 3 o 4."

    correct_idx = q["correct_index"]
    correct_term = q["options"][correct_idx]
    respuesta = ""

    if choice == correct_idx:
        respuesta = "✅ ¡Muy bien! Elegiste el término correcto 💪\n"
    else:
        respuesta = (
            "❌ No pasa nada, equivocarse también es parte de aprender.\n"
            f"La respuesta correcta era la opción {correct_idx + 1}: «{correct_term}».\n"
        )

    current_index += 1
    awaiting_answer = False

    return respuesta + "\n" + flashcards_next_question()


def handle_flashcards(message):
    if awaiting_answer:
        return handle_flashcards_answer(message)
    else:
        # por si se pierde el estado
        return flashcards_next_question()


# ====================
# MODE 2: QUIZ TIPO EXAMEN (término → elegir definición)
# ====================

def quiz_next_question():
    global current_index, awaiting_answer, last_question

    if current_index >= len(quiz_questions):
        reset_state()
        return "¡Terminamos todas las preguntas tipo examen de este módulo! 🌟\n\n" + menu_text()

    q = quiz_questions[current_index]
    last_question = q
    awaiting_answer = True

    texto = [
        f"📖 Pregunta tipo examen {current_index + 1}/{len(quiz_questions)}",
        "",
        f"Según el manual, ¿cuál definición corresponde al término:",
        f"👉 {q['term']}",
        ""
    ]
    for i, opt in enumerate(q["options"], start=1):
        texto.append(f"{i}) {opt}")

    texto.append("\nEscribí el número de tu respuesta (o «salir» para volver al menú).")
    return "\n".join(texto)


def handle_quiz_answer(message):
    global current_index, awaiting_answer

    msg = message.strip().lower()
    if msg == "salir":
        reset_state()
        return "Volvemos al menú principal 😉\n\n" + menu_text()

    if not msg.isdigit():
        return "Recordá que acá solo espero un número (1, 2, 3, 4) 😊"

    choice = int(msg) - 1
    q = last_question
    if choice < 0 or choice >= len(q["options"]):
        return "Ese número está fuera de rango. Probá de nuevo con 1, 2, 3 o 4."

    correct_idx = q["correct_index"]
    correct_def = q["options"][correct_idx]
    respuesta = ""

    if choice == correct_idx:
        respuesta = "✅ ¡Excelente! Esa es la definición correcta. Vas muy bien 💪\n"
    else:
        respuesta = (
            "❌ Esta vez no fue, pero estás practicando y eso ya es un paso enorme 👏\n"
            f"La opción correcta era la {correct_idx + 1}:\n{correct_def}\n"
        )

    current_index += 1
    awaiting_answer = False

    return respuesta + "\n" + quiz_next_question()


def handle_quiz(message):
    if awaiting_answer:
        return handle_quiz_answer(message)
    else:
        return quiz_next_question()


# ====================
# MODE 3: VERDADERO / FALSO
# ====================

def vf_next_question():
    global current_index, awaiting_answer, last_question

    if current_index >= len(vf_questions):
        reset_state()
        return "¡Terminamos las afirmaciones de Verdadero/Falso! 🎯\n\n" + menu_text()

    q = vf_questions[current_index]
    last_question = q
    awaiting_answer = True

    texto = [
        f"🔎 Verdadero o Falso {current_index + 1}/{len(vf_questions)}",
        "",
        q["statement"],
        "",
        "Respondé con V (verdadero) o F (falso). También podés escribir «salir» para volver al menú."
    ]
    return "\n".join(texto)


def handle_vf_answer(message):
    global current_index, awaiting_answer

    msg = message.strip().lower()
    if msg == "salir":
        reset_state()
        return "Volvemos al menú principal 😉\n\n" + menu_text()

    if msg in ["v", "verdadero"]:
        user_true = True
    elif msg in ["f", "falso"]:
        user_true = False
    else:
        return "Acá espero que respondas con V (verdadero) o F (falso) 😊"

    q = last_question
    is_true = q["is_true"]
    explanation = q["explanation"]
    respuesta = ""

    if user_true == is_true:
        respuesta = (
            "✅ ¡Muy bien! Tu respuesta es correcta 🎉\n"
            f"{explanation}\n"
        )
    else:
        respuesta = (
            "❌ Esta vez no coincidió, pero justo por eso estas actividades son útiles.\n"
            f"{explanation}\n"
        )

    current_index += 1
    awaiting_answer = False

    return respuesta + "\n" + vf_next_question()


def handle_vf(message):
    if awaiting_answer:
        return handle_vf_answer(message)
    else:
        return vf_next_question()


# ====================
# RUTAS FLASK
# ====================

@app.route("/")
def index():
    return render_template("chat.html")


@app.route("/upload_manual", methods=["POST"])
def upload_manual():
    """
    Recibe un PDF, extrae el texto, construye flashcards y preguntas.
    """
    global manual_text, flashcards_cards, flashcards_mc, quiz_questions, vf_questions

    if "manual" not in request.files:
        return jsonify({"ok": False, "message": "No se encontró el archivo. ¿Probás de nuevo?"})

    file = request.files["manual"]

    if file.filename == "":
        return jsonify({"ok": False, "message": "No seleccionaste ningún archivo."})

    if not file.filename.lower().endswith(".pdf"):
        return jsonify({"ok": False, "message": "Por ahora solo acepto archivos PDF 🧾."})

    try:
        temp_path = "manual_subido.pdf"
        file.save(temp_path)
        text = extract_text_from_pdf(temp_path)

        if not text.strip():
            return jsonify({
                "ok": False,
                "message": "No pude leer texto en ese PDF. Revisá que no sea solo una imagen."
            })

        manual_text = text

        # Construir estructuras
        flashcards_cards = build_flashcards_from_text(text)
        flashcards_mc = build_flashcards_mc_from_cards(flashcards_cards)
        quiz_questions = build_quiz_from_flashcards(flashcards_cards)
        vf_questions = build_vf_from_cards(flashcards_cards)

        reset_state()

        extra = ""
        if not flashcards_cards:
            extra = (
                "\n\nNo encontré líneas con formato «Término: definición». "
                "Las actividades automáticas pueden ser limitadas con este archivo."
            )

        return jsonify({
            "ok": True,
            "message": (
                "Listo ✨ Ya leí tu manual y armé actividades.\n"
                f"• Flashcards con opción múltiple: {len(flashcards_mc)}\n"
                f"• Preguntas tipo examen: {len(quiz_questions)}\n"
                f"• Verdadero/Falso: {len(vf_questions)}\n\n"
                + menu_text()
                + extra
            )
        })

    except Exception:
        print("ERROR AL CARGAR PDF:")
        print(traceback.format_exc())
        return jsonify({
            "ok": False,
            "message": "Ocurrió un problema leyendo el PDF. Probá con otro archivo o avisale al profe."
        })


@app.route("/api/message", methods=["POST"])
def api_message():
    """
    Maneja los mensajes del alumno sin usar IA.
    Solo números/palabras simples, guiando el estudio.
    """
    data = request.get_json() or {}
    user_message = (data.get("message") or "").strip()

    if not user_message:
        return jsonify({"reply": "Escribime algo cortito así seguimos 😊"})

    if not manual_text:
        return jsonify({
            "reply": (
                "Todavía no cargaste ningún manual 📚.\n"
                "Subí el PDF del módulo que quieras repasar y después escribime de nuevo."
            )
        })

    global current_mode

    if current_mode is None:
        reset_state()
        reply = menu_text()
    elif current_mode == "menu":
        reply = handle_menu(user_message)
    elif current_mode == "flashcards":
        reply = handle_flashcards(user_message)
    elif current_mode == "quiz":
        reply = handle_quiz(user_message)
    elif current_mode == "vf":
        reply = handle_vf(user_message)
    else:
        reset_state()
        reply = menu_text()

    return jsonify({"reply": reply})


if __name__ == "__main__":
    app.run(debug=True)