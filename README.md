
import os
import random
import sqlite3
import asyncio
import discord

from discord.ext import commands
from discord import app_commands

NOME = "Quiz ETERNO DEUS"
TOKEN = os.getenv("DISCORD_TOKEN")
BANCO = "quiz_eterno_deus.db"

intents = discord.Intents.default()
bot = commands.Bot(command_prefix="!", intents=intents)

partidas = {}
pontos_dificuldade = {
    "facil": 10,
    "medio": 20,
    "dificil": 30
}

PERGUNTAS = [
    {
        "pergunta": "Quem construiu a arca por ordem de Deus?",
        "opcoes": ["Moisés", "Noé", "Abraão", "Davi"],
        "resposta": 1,
        "categoria": "Antigo Testamento",
        "dificuldade": "facil",
        "referencia": "Gênesis 6:13-22"
    },
    {
        "pergunta": "Quem derrotou Golias?",
        "opcoes": ["Saul", "Salomão", "Davi", "Josué"],
        "resposta": 2,
        "categoria": "Pessoas bíblicas",
        "dificuldade": "facil",
        "referencia": "1 Samuel 17"
    },
    {
        "pergunta": "Em qual cidade Jesus nasceu?",
        "opcoes": ["Nazaré", "Jerusalém", "Belém", "Cafarnaum"],
        "resposta": 2,
        "categoria": "Novo Testamento",
        "dificuldade": "facil",
        "referencia": "Mateus 2:1"
    },
    {
        "pergunta": "Quantos apóstolos Jesus escolheu?",
        "opcoes": ["7", "10", "12", "40"],
        "resposta": 2,
        "categoria": "Novo Testamento",
        "dificuldade": "facil",
        "referencia": "Lucas 6:13"
    },
    {
        "pergunta": "Quem recebeu os Dez Mandamentos?",
        "opcoes": ["Arão", "Moisés", "Elias", "Isaías"],
        "resposta": 1,
        "categoria": "Antigo Testamento",
        "dificuldade": "facil",
        "referencia": "Êxodo 31:18"
    },
    {
        "pergunta": "Qual foi o primeiro milagre de Jesus no Evangelho de João?",
        "opcoes": [
            "Curar um cego",
            "Multiplicar pães",
            "Transformar água em vinho",
            "Andar sobre o mar"
        ],
        "resposta": 2,
        "categoria": "Ensinamentos de Jesus",
        "dificuldade": "medio",
        "referencia": "João 2:1-11"
    },
    {
        "pergunta": "Quem foi engolido por um grande peixe?",
        "opcoes": ["Jonas", "Jó", "Jeremias", "Ezequiel"],
        "resposta": 0,
        "categoria": "Pessoas bíblicas",
        "dificuldade": "facil",
        "referencia": "Jonas 1:17"
    },
    {
        "pergunta": "Qual é o último livro da Bíblia?",
        "opcoes": ["Judas", "Hebreus", "Apocalipse", "Atos"],
        "resposta": 2,
        "categoria": "Novo Testamento",
        "dificuldade": "facil",
        "referencia": "Apocalipse"
    },
    {
        "pergunta": "Quem interpretou os sonhos do faraó?",
        "opcoes": ["José", "Daniel", "Samuel", "Neemias"],
        "resposta": 0,
        "categoria": "Pessoas bíblicas",
        "dificuldade": "medio",
        "referencia": "Gênesis 41"
    },
    {
        "pergunta": "Quem sucedeu Moisés na liderança de Israel?",
        "opcoes": ["Calebe", "Josué", "Arão", "Samuel"],
        "resposta": 1,
        "categoria": "Antigo Testamento",
        "dificuldade": "medio",
        "referencia": "Josué 1:1-2"
    }
]


# ---------- BANCO DE DADOS ----------

def preparar_banco():
    with sqlite3.connect(BANCO) as db:
        db.execute("""
            CREATE TABLE IF NOT EXISTS jogadores (
                usuario TEXT PRIMARY KEY,
                nome TEXT NOT NULL,
                pontos INTEGER DEFAULT 0,
                acertos INTEGER DEFAULT 0,
                respondidas INTEGER DEFAULT 0
            )
        """)


def salvar_resultado(usuario, nome, pontos, acertou):
    with sqlite3.connect(BANCO) as db:
        db.execute("""
            INSERT INTO jogadores
                (usuario, nome, pontos, acertos, respondidas)
            VALUES (?, ?, ?, ?, 1)
            ON CONFLICT(usuario) DO UPDATE SET
                nome = excluded.nome,
                pontos = jogadores.pontos + excluded.pontos,
                acertos = jogadores.acertos + excluded.acertos,
                respondidas = jogadores.respondidas + 1
        """, (
            str(usuario),
            nome,
            pontos,
            int(acertou)
        ))


# ---------- BOTÕES DE RESPOSTA ----------

class Respostas(discord.ui.View):
    def __init__(self, usuario, pergunta, dificuldade, tempo):
        super().__init__(timeout=tempo)
        self.usuario = usuario
        self.pergunta = pergunta
        self.dificuldade = dificuldade
        self.respondida = False

        for indice, opcao in enumerate(pergunta["opcoes"]):
            botao = discord.ui.Button(
                label=f"{chr(65 + indice)}. {opcao}",
                style=discord.ButtonStyle.primary
            )
            botao.callback = self.criar_callback(indice)
            self.add_item(botao)

    def criar_callback(self, escolha):
        async def callback(interaction: discord.Interaction):
            if interaction.user.id != self.usuario:
                await interaction.response.send_message(
                    "Esta pergunta pertence a outro jogador!",
                    ephemeral=True
                )
                return

            if self.respondida:
                await interaction.response.send_message(
                    "Você já respondeu a esta pergunta.",
                    ephemeral=True
                )
                return

            self.respondida = True
            for item in self.children:
                item.disabled = True

            acertou = escolha == self.pergunta["resposta"]
            pontos = (
                pontos_dificuldade[self.dificuldade]
                if acertou else 0
            )

            salvar_resultado(
                interaction.user.id,
                interaction.user.display_name,
                pontos,
                acertou
            )

            correta = self.pergunta["opcoes"][
                self.pergunta["resposta"]
            ]

            if acertou:
                mensagem = f"✅ Resposta correta! Você ganhou **{pontos} pontos**."
            else:
                mensagem = f"❌ Resposta incorreta. A resposta era **{correta}**."

            mensagem += (
                f"\n📖 Referência: {self.pergunta['referencia']}"
            )

            partidas.pop(self.usuario, None)

            await interaction.response.edit_message(
                content=mensagem,
                embed=None,
                view=self
            )

            self.stop()

        return callback

    async def on_timeout(self):
        if not self.respondida:
            for item in self.children:
                item.disabled = True

            partidas.pop(self.usuario, None)


# ---------- EVENTOS ----------

@bot.event
async def on_ready():
    try:
        comandos = await bot.tree.sync()
        print(f"{NOME} está online como {bot.user}!")
        print(f"{len(comandos)} comandos sincronizados.")
    except Exception as erro:
        print(f"Erro ao sincronizar comandos: {erro}")


# ---------- INICIAR QUIZ ----------

@bot.tree.command(
    name="quiz_iniciar",
    description="Começar um quiz bíblico"
)
@app_commands.choices(dificuldade=[
    app_commands.Choice(name="Fácil", value="facil"),
    app_commands.Choice(name="Médio", value="medio"),
    app_commands.Choice(name="Difícil", value="dificil"),
    app_commands.Choice(name="Aleatório", value="aleatorio")
])
@app_commands.describe(
    dificuldade="Escolha a dificuldade",
    tempo="Segundos para responder"
)
async def quiz_iniciar(
    interaction: discord.Interaction,
    dificuldade: app_commands.Choice[str],
    tempo: app_commands.Range[int, 10, 120] = 30
):
    usuario = interaction.user.id

    if usuario in partidas:
        await interaction.response.send_message(
            "Você já tem um quiz ativo!",
            ephemeral=True
        )
        return

    nivel = dificuldade.value

    if nivel == "aleatorio":
        nivel = random.choice(list(pontos_dificuldade.keys()))

    disponiveis = [
        p for p in PERGUNTAS
        if p["dificuldade"] == nivel
    ]

    pergunta = random.choice(disponiveis)
    partidas[usuario] = True

    embed = discord.Embed(
        title=f"📖 {NOME}",
        description=(
            f"**{pergunta['pergunta']}**\n\n"
            + "\n".join(
                f"**{chr(65+i)}.** {opcao}"
                for i, opcao in enumerate(pergunta["opcoes"])
            )
        ),
        color=discord.Color.gold()
    )

    embed.add_field(
        name="Dificuldade",
        value=nivel.capitalize(),
        inline=True
    )
    embed.add_field(
        name="Tempo",
        value=f"{tempo} segundos",
        inline=True
    )
    embed.add_field(
        name="Prêmio",
        value=f"{pontos_dificuldade[nivel]} pontos",
        inline=True
    )

    view = Respostas(usuario, pergunta, nivel, tempo)

    await interaction.response.send_message(
        embed=embed,
        view=view
    )

    async def encerrar():
        await asyncio.sleep(tempo)

        if usuario in partidas and not view.respondida:
            partidas.pop(usuario, None)

            for item in view.children:
                item.disabled = True

            try:
                await interaction.edit_original_response(
                    content="⏰ Tempo esgotado! Tente novamente.",
                    view=view
                )
            except discord.HTTPException:
                pass

    asyncio.create_task(encerrar())


# ---------- RANKING ----------

@bot.tree.command(
    name="quiz_placar",
    description="Ver o ranking dos jogadores"
)
async def quiz_placar(interaction: discord.Interaction):
    with sqlite3.connect(BANCO) as db:
        jogadores = db.execute("""
            SELECT nome, pontos, acertos
            FROM jogadores
            ORDER BY pontos DESC, acertos DESC
            LIMIT 10
        """).fetchall()

    if not jogadores:
        await interaction.response.send_message(
            "Ainda não há jogadores no ranking!"
        )
        return

    linhas = [
        f"**{i}.** {nome} — 🏆 {pontos} pontos | "
        f"✅ {acertos} acertos"
        for i, (nome, pontos, acertos)
        in enumerate(jogadores, start=1)
    ]

    embed = discord.Embed(
        title=f"🏆 Ranking — {NOME}",
        description="\n".join(linhas),
        color=discord.Color.gold()
    )

    await interaction.response.send_message(embed=embed)


# ---------- PERFIL ----------

@bot.tree.command(
    name="quiz_perfil",
    description="Ver suas estatísticas"
)
async def quiz_perfil(interaction: discord.Interaction):
    with sqlite3.connect(BANCO) as db:
        jogador = db.execute("""
            SELECT nome, pontos, acertos, respondidas
            FROM jogadores
            WHERE usuario = ?
        """, (str(interaction.user.id),)).fetchone()

    if not jogador:
        await interaction.response.send_message(
            "Você ainda não jogou. Use /quiz_iniciar!",
            ephemeral=True
        )
        return

    nome, pontos, acertos, respondidas = jogador
    taxa = acertos / respondidas * 100 if respondidas else 0

    embed = discord.Embed(
        title=f"📊 Perfil de {nome}",
        color=discord.Color.blurple()
    )
    embed.add_field(name="Pontos", value=str(pontos))
    embed.add_field(name="Acertos", value=str(acertos))
    embed.add_field(name="Respondidas", value=str(respondidas))
    embed.add_field(name="Taxa de acerto", value=f"{taxa:.1f}%")

    await interaction.response.send_message(
        embed=embed,
        ephemeral=True
    )


# ---------- AJUDA ----------

@bot.tree.command(
    name="quiz_ajuda",
    description="Mostrar os comandos disponíveis"
)
async def quiz_ajuda(interaction: discord.Interaction):
    embed = discord.Embed(
        title=f"📘 Ajuda — {NOME}",
        description=(
            "`/quiz_iniciar` — iniciar uma pergunta\n"
            "`/quiz_placar` — ver o ranking\n"
            "`/quiz_perfil` — ver suas estatísticas\n"
            "`/quiz_teste` — testar o bot\n"
            "`/quiz_parar` — cancelar sua pergunta\n"
            "`/quiz_ajuda` — mostrar esta ajuda"
        ),
        color=discord.Color.blue()
    )
    await interaction.response.send_message(
        embed=embed,
        ephemeral=True
    )


# ---------- TESTE ----------

@bot.tree.command(
    name="quiz_teste",
    description="Verificar se o bot está online"
)
async def quiz_teste(interaction: discord.Interaction):
    await interaction.response.send_message(
        f"✅ **{NOME} está online!**",
        ephemeral=True
    )


# ---------- PARAR ----------

@bot.tree.command(
    name="quiz_parar",
    description="Cancelar sua pergunta ativa"
)
async def quiz_parar(interaction: discord.Interaction):
    if interaction.user.id not in partidas:
        await interaction.response.send_message(
            "Você não tem uma pergunta ativa.",
            ephemeral=True
        )
        return

    partidas.pop(interaction.user.id, None)

    await interaction.response.send_message(
        "Participação cancelada.",
        ephemeral=True
    )


# ---------- EXECUÇÃO ----------

if not TOKEN:
    raise RuntimeError(
        "Configure a variável de ambiente DISCORD_TOKEN."
    )

preparar_banco()
bot.run(TOKEN)
