import os
import re
import aiohttp
import discord
from dotenv import load_dotenv

load_dotenv()

DISCORD_TOKEN = os.getenv("DISCORD_TOKEN")
FINNHUB_API_KEY = os.getenv("FINNHUB_API_KEY")
QUOTE_CHANNEL = os.getenv("QUOTE_CHANNEL", "").lower()

# Match: #AAPL | #BRK.B | #BTC | #BINANCE:BTCUSDT
TICKER_RE = re.compile(
    r"^\s*#([A-Za-z]{1,15}(?::[A-Za-z0-9]{3,20})?(?:[.\-][A-Za-z0-9]{1,10})?)\s*$"
)

CRYPTO_ALIASES = {
    "BTC": "BINANCE:BTCUSDT",
    "ETH": "BINANCE:ETHUSDT",
    "SOL": "BINANCE:SOLUSDT",
    "XRP": "BINANCE:XRPUSDT",
    "ADA": "BINANCE:ADAUSDT",
    "DOGE": "BINANCE:DOGEUSDT",
    "AVAX": "BINANCE:AVAXUSDT",
    "LINK": "BINANCE:LINKUSDT",
    "MATIC": "BINANCE:MATICUSDT",
    "BNB": "BINANCE:BNBUSDT",
}

FINNHUB_BASE = "https://finnhub.io/api/v1"


def resolve_symbol(raw):
    s = raw.upper()
    if ":" in s:
        return s, "crypto"
    if s in CRYPTO_ALIASES:
        return CRYPTO_ALIASES[s], "crypto"
    return s, "stock"


async def finnhub_get(session, path, params):
    params["token"] = FINNHUB_API_KEY
    async with session.get(f"{FINNHUB_BASE}{path}", params=params) as r:
        return await r.json()


intents = discord.Intents.default()
intents.message_content = True
client = discord.Client(intents=intents)
session = aiohttp.ClientSession()


@client.event
async def on_ready():
    print(f"Bot logged in as {client.user}")


@client.event
async def on_message(message):
    if message.author.bot:
        return

    if QUOTE_CHANNEL and message.channel.name.lower() != QUOTE_CHANNEL:
        return

    match = TICKER_RE.match(message.content)
    if not match:
        return

    raw = match.group(1)
    symbol, asset_type = resolve_symbol(raw)

    quote = await finnhub_get(session, "/quote", {"symbol": symbol})

    price = quote.get("c")
    change = quote.get("d")
    percent = quote.get("dp")
    high = quote.get("h")
    low = quote.get("l")
    open_ = quote.get("o")
    prev = quote.get("pc")

    if not price:
        await message.reply("Invalid ticker.")
        return

    embed = discord.Embed(
        title=f"{symbol} ({'Crypto' if asset_type == 'crypto' else 'Stock'})",
        description=f"**Price:** {price}\n**Change:** {change} ({percent}%)",
        color=0x00ff99,
    )

    embed.add_field(name="Open", value=open_, inline=True)
    embed.add_field(name="High", value=high, inline=True)
    embed.add_field(name="Low", value=low, inline=True)
    embed.add_field(name="Prev Close", value=prev, inline=True)

    await message.reply(embed=embed)


client.run(DISCORD_TOKEN)
