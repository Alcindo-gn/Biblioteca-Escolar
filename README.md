# Biblioteca-Escolar
Sistema de Biblioteca Escolar - Controla empréstimos, devoluções, atrasos e multas. Feito em Python + SQLite, integrando Banco de Dados, Lógica Computacional e Introdução à Programação.
import sqlite3
from datetime import date, timedelta

# --- BD: Criação das tabelas ---
conn = sqlite3.connect("biblioteca.db")
cursor = conn.cursor()

cursor.execute("CREATE TABLE IF NOT EXISTS alunos (id INTEGER PRIMARY KEY, nome TEXT, turma TEXT)")
cursor.execute("CREATE TABLE IF NOT EXISTS livros (id INTEGER PRIMARY KEY, titulo TEXT, autor TEXT, disponivel INTEGER DEFAULT 1)")
cursor.execute("CREATE TABLE IF NOT EXISTS emprestimos (id INTEGER PRIMARY KEY, aluno_id INTEGER, livro_id INTEGER, data_emprestimo TEXT, data_devolucao TEXT, devolvido INTEGER DEFAULT 0)")

# Livros de exemplo (só cadastra se estiver vazio)
if cursor.execute("SELECT COUNT(*) FROM livros").fetchone()[0] == 0:
    cursor.executemany("INSERT INTO livros (titulo, autor) VALUES (?, ?)", [
        ("Dom Casmurro", "Machado de Assis"),
        ("Harry Potter", "J.K. Rowling"),
        ("O Pequeno Príncipe", "Saint-Exupéry"),
    ])
conn.commit()

# --- LC + IP: Funções ---
def cadastrar_aluno():
    nome = input("Nome do aluno: ")
    turma = input("Turma: ")
    cursor.execute("INSERT INTO alunos (nome, turma) VALUES (?, ?)", (nome, turma))
    conn.commit()
    print("Aluno cadastrado!")

def ver_livros():
    print("\n--- LIVROS ---")
    for id, titulo, autor, disp in cursor.execute("SELECT * FROM livros"):
        status = "Disponível" if disp == 1 else "Emprestado"
        print(f"{id} - {titulo} ({autor}) -> {status}")

def emprestar():
    aluno_id = int(input("ID do aluno: "))
    livro_id = int(input("ID do livro: "))

    # LC: SE aluno tem atraso ENTÃO bloqueia
    atraso = cursor.execute("SELECT COUNT(*) FROM emprestimos WHERE aluno_id=? AND devolvido=0 AND data_devolucao < ?",
                            (aluno_id, str(date.today()))).fetchone()[0]
    if atraso > 0:
        print("BLOQUEADO: Aluno com devolução atrasada!")
        return

    # LC: SE livro = disponível ENTÃO empresta
    livro = cursor.execute("SELECT disponivel FROM livros WHERE id=?", (livro_id,)).fetchone()
    if livro and livro[0] == 1:
        hoje = date.today()
        devolucao = hoje + timedelta(days=7)
        cursor.execute("INSERT INTO emprestimos (aluno_id, livro_id, data_emprestimo, data_devolucao) VALUES (?, ?, ?, ?)",
                       (aluno_id, livro_id, str(hoje), str(devolucao)))
        cursor.execute("UPDATE livros SET disponivel=0 WHERE id=?", (livro_id,))
        conn.commit()
        print(f"Empréstimo feito! Devolver até {devolucao}")
    else:
        print("Livro não disponível!")

def devolver():
    livro_id = int(input("ID do livro para devolver: "))
    emp = cursor.execute("SELECT id, data_devolucao FROM emprestimos WHERE livro_id=? AND devolvido=0", (livro_id,)).fetchone()
    if not emp:
        print("Este livro não está emprestado.")
        return

    # LC: SE dias de atraso > 0 ENTÃO multa = dias x R$ 1,00
    data_dev = date.fromisoformat(emp[1])
    dias_atraso = (date.today() - data_dev).days
    multa = dias_atraso * 1.00 if dias_atraso > 0 else 0

    cursor.execute("UPDATE emprestimos SET devolvido=1 WHERE id=?", (emp[0],))
    cursor.execute("UPDATE livros SET disponivel=1 WHERE id=?", (livro_id,))
    conn.commit()
    print(f"Devolvido! Atraso: {max(0, dias_atraso)} dias | Multa: R$ {multa:.2f}")

# --- IP: Menu principal ---
while True:
    print("\n=== BIBLIOTECA ESCOLAR ===")
    print("1 - Cadastrar aluno")
    print("2 - Ver livros")
    print("3 - Emprestar livro")
    print("4 - Devolver livro")
    print("0 - Sair")
    opcao = input("Escolha: ")

    if opcao == "1": cadastrar_aluno()
    elif opcao == "2": ver_livros()
    elif opcao == "3": emprestar()
    elif opcao == "4": devolver()
    elif opcao == "0": break
    else: print("Opção inválida!")

print("Até logo!")
conn.close()
