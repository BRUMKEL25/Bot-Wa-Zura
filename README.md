import discord
from discord.ext import commands
from datetime import datetime, timedelta
import math
import os

TOKEN = os.getenv("DISCORD_TOKEN")  # Ambil token dari environment variable

bot = commands.Bot(command_prefix="!", intents=discord.Intents.all())

# =======================
# Data user (sementara)
# =======================
users_data = {}  # {user_id: {"exp":..., "level":..., "last_msg":..., "joined_at":...}}

EXP_PER_WORD = 20
COOLDOWN = timedelta(seconds=5)
BASE_EXP = 500  # Level 0 -> 1

# =======================
# Fungsi Hitung EXP untuk level selanjutnya
# =======================
def next_level_exp(level):
    exp_needed = BASE_EXP * (1.8 ** level)
    return math.floor(exp_needed)

# =======================
# Event: Bot siap
# =======================
@bot.event
async def on_ready():
    print(f"Bot online sebagai {bot.user}")

# =======================
# Event: Pesan Masuk (untuk EXP)
# =======================
@bot.event
async def on_message(message):
    if message.author.bot:
        return

    user_id = message.author.id
    now = datetime.utcnow()

    if user_id not in users_data:
        users_data[user_id] = {
            "exp": 0,
            "level": 0,
            "last_msg": now - COOLDOWN,
            "joined_at": message.author.joined_at
        }

    user = users_data[user_id]

    if now - user["last_msg"] >= COOLDOWN:
        word_count = len(message.content.split())
        gained_exp = word_count * EXP_PER_WORD
        user["exp"] += gained_exp
        user["last_msg"] = now

        exp_needed = next_level_exp(user["level"])
        if user["exp"] >= exp_needed:
            user["level"] += 1
            user["exp"] -= exp_needed
            await message.channel.send(
                f"🎉 Selamat {message.author.mention}! Kamu naik ke level {user['level']}!"
            )

    await bot.process_commands(message)

# =======================
# Command !status atau !stat
# =======================
@bot.command(aliases=["stat"])
async def status(ctx, member: discord.Member = None):
    if member is None:
        member = ctx.author

    user_id = member.id
    if user_id not in users_data:
        users_data[user_id] = {
            "exp": 0,
            "level": 0,
            "last_msg": datetime.utcnow() - COOLDOWN,
            "joined_at": member.joined_at
        }

    user = users_data[user_id]

    join_time = member.joined_at
    now = datetime.utcnow()
    duration = now - join_time
    days = duration.days
    hours, remainder = divmod(duration.seconds, 3600)
    minutes, _ = divmod(remainder, 60)

    message_text = (
        f"👤 Mention: {member.mention}\n"
        f"📝 Nama: {member.name}\n"
        f"⏱ Lama di server: {days} hari, {hours} jam, {minutes} menit\n"
        f"⭐ Level: {user['level']}\n"
        f"💠 EXP: {user['exp']}/{next_level_exp(user['level'])}"
    )

    await ctx.send(message_text)

# =======================
# Jalankan Bot
# =======================
bot.run(TOKEN)
