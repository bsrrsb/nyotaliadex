"""Nyotaliadex Discord bot.

The bot periodically spawns a character in a text channel. Users can use
``!catch <character name>`` during the catch window and inspect their own
collection with ``!collection``. Catch guesses use the ``Nyo!Country`` format,
such as ``!catch Nyo!Japan``.

Configuration is read from environment variables. Never commit a Discord
token to source control.
"""

from __future__ import annotations

import asyncio
import json
import logging
import os
import random
import unicodedata
from pathlib import Path
from typing import Any

import discord
from discord.ext import commands


logging.basicConfig(
    level=os.getenv("LOG_LEVEL", "INFO").upper(),
    format="%(asctime)s %(levelname)s %(name)s: %(message)s",
)
logger = logging.getLogger("nyotaliadex")


def env_bool(name: str, default: bool) -> bool:
    """Read a boolean environment variable with a useful validation error."""
    value = os.getenv(name)
    if value is None:
        return default

    normalized = value.strip().lower()

    if normalized in {"1", "true", "yes", "on"}:
        return True

    if normalized in {"0", "false", "no", "off"}:
        return False

    raise ValueError(
        f"{name} must be true/false (or 1/0), but received {value!r}"
    )


def env_int(
    name: str,
    default: int,
    *,
    minimum: int | None = None,
) -> int:
    """Read and validate an integer environment variable."""
    value = os.getenv(name)

    if value is None or not value.strip():
        result = default
    else:
        try:
            result = int(value)
        except ValueError as exc:
            raise ValueError(f"{name} must be an integer") from exc

    if minimum is not None and result < minimum:
        raise ValueError(f"{name} must be at least {minimum}")

    return result


def load_saved_channel_id(path: Path) -> int | None:
    """Load a channel selected with !config, if one has been saved."""
    if not path.exists():
        return None

    try:
        raw = json.loads(path.read_text(encoding="utf-8"))
    except (OSError, json.JSONDecodeError) as exc:
        logger.warning("Could not load %s: %s; ignoring it", path, exc)
        return None

    channel_id = raw.get("channel_id") if isinstance(raw, dict) else None

    if isinstance(channel_id, int) and channel_id > 0:
        return channel_id

    return None


def load_settings() -> dict[str, Any]:
    token = os.getenv("DISCORD_TOKEN", "").strip()

    if not token:
        raise RuntimeError(
            "DISCORD_TOKEN is not set. Add the new Discord bot token as a "
            "Replit Secret or environment variable before starting the bot."
        )

    settings_file = Path(
        os.getenv("BOT_SETTINGS_FILE", "data/bot_settings.json")
    )

    env_channel_id = env_int("SPAWN_CHANNEL_ID", 0, minimum=0)
    channel_id = load_saved_channel_id(settings_file) or env_channel_id

    return {
        "token": token,
        "channel_id": channel_id or None,
        "spawn_interval": env_int("SPAWN_INTERVAL", 1200, minimum=1),
        "catch_window": env_int("CATCH_WINDOW", 60, minimum=1),
        "first_come_first_served": env_bool(
            "FIRST_COME_FIRST_SERVED",
            True,
        ),
        "use_mix": env_bool("USE_MIX", True),
        "prefix": os.getenv("COMMAND_PREFIX", "!") or "!",
        "data_file": Path(
            os.getenv("COLLECTIONS_FILE", "data/collections.json")
        ),
        "settings_file": settings_file,
    }


NYO_CHARS = [
    {"name": "Nyo!Italy", "code": "#97D1", "hp": -9, "atk": -16, "rarity": "Common"},
    {"name": "Nyo!Germany", "code": "#7A2E", "hp": -5, "atk": -12, "rarity": "Common"},
    {"name": "Nyo!Japan", "code": "#B23B", "hp": -7, "atk": -14, "rarity": "Common"},
    {"name": "Nyo!America", "code": "#C44F", "hp": -11, "atk": -8, "rarity": "Uncommon"},
    {"name": "Nyo!Russia", "code": "#6E3A", "hp": -13, "atk": -10, "rarity": "Uncommon"},
    {"name": "Nyo!England", "code": "#5D6E", "hp": -8,, "atk": -15,, "rarity": "Rare"},
    {"name": "Nyo!France", "code": "#8A5C", "hp": -6,, "atk": -11,, "rarity": "Rare"},
    {"name": "Nyo!China", "code": "#D24B", "hp": -10,, "atk": -13,, "rarity": "Rare"},
]

MIX_CHARS = [
    {"name": "Nyo!Ukraine", "code": "#4F8A", "hp": -12,, "atk": -9,, "rarity": "Ultra-Rare"},
    {"name": "Nyo!Belarus", "code": "#6A4F", "hp": -14,, "atk": -7,, "rarity": "Ultra-Rare"},
    {"name": "Nyo!Spain", "code": "#C85A", "hp": -9,, "atk": -11,, "rarity": "Ultra-Rare"},
    {"name": "Nyo!Greece", "code": "#3B7E", "hp": -8,, "atk": -12,, "rarity": "Legendary"},
    {"name": "Nyo!Türkiye", "code": "#D2691E", "hp": -10,, "atk": -14,, "rarity": "Legendary"},
    {"name": "Nyo!Egypt", "code": "#C19A6B", "hp": -11,, "atk": -13,, "rarity": "Legendary"},
]


def canonical_character_name(name: str) -> str:
    """Convert legacy character labels to the current Nyo!Country format."""
    if name.startswith("Nyo!"):
        return name

    if name == "Nyotalia":
        return "Nyo!Italy"

    if name.startswith("Nyotalia "):
        return f"Nyo!{name.removeprefix('Nyotalia ')}"

    return name


def catch_name(character: dict[str, Any]) -> str:
    """Return the short name users must guess for a character."""
    return canonical_character_name(character["name"])


def normalize_catch_input(value: str) -> str:
    """Compare names case-insensitively and without accent differences."""
    decomposed = unicodedata.normalize("NFKD", value.casefold())

    return "".join(
        character
        for character in decomposed
        if not unicodedata.combining(character)
    )


def catch_name_aliases(character: dict[str, Any]) -> set[str]:
    """Return all accepted spellings for a character's catch name."""
    canonical = catch_name(character)
    country = canonical.removeprefix("Nyo!")

    return {
        normalize_catch_input(canonical),
        normalize_catch_input(f"Nyo {country}"),
        normalize_catch_input(country),
    }


class NyotaliadexBot(commands.Bot):
    """Discord bot with one managed spawn loop and atomic catch operations."""

    def __init__(self, settings: dict[str, Any]) -> None:
        intents = discord.Intents.default()
        intents.message_content = True

        super().__init__(
            command_prefix=settings["prefix"],
            intents=intents,
            help_command=commands.DefaultHelpCommand(
                no_category="Commands",
            ),
        )

        self.settings = settings

        self.characters = NYO_CHARS + (
            MIX_CHARS if settings["use_mix"] else []
        )

        self.spawned: dict[str, Any] | None = None
        self.spawn_message_channel: discord.TextChannel | None = None
        self.caught_by: set[int] = set()
        self.collections: dict[str, set[str]] = self.load_collections()
        self.pending_trades: dict[int, dict[str, Any]]] = {}
        self.state_lock = asyncio.Lock()
        self.spawn_task:asyncio.Task[None] | None = None

    def load_collections(self) -> dict[str, set[str]]:
        path: Path = self.settings["data_file"]

        if not path.exists():
            return {}

        try:
            raw = json.loads(path.read_text(encoding="utf-8"))
        except (OSError, json.JSONDecodeError) as exc:
            logger.warning(
                "Could not load %s: %s; starting empty",
                path,
                exc,
            )
            return {}

        if not isinstance(raw, dict):
            logger.warning("Ignoring invalid collection data in %s", path)
            return {}

        return {
            str(user_id): {
                canonical_character_name(str(name))
                for namein names
                if isinstance(name, str)
            }
            for user_id, names in raw.items()
            if isinstance(names, list)
        }

    def save_collections(self) -> None:
        path: Path = self.settings["data_file"]
        path.parent.mkdir(parents=True, exist_ok=True)

        payload = {
            user_id: sorted(names)
            for user_id, names in self.collections.items()
        }

        temporary_path = path.with_suffix(f"{path.suffix}.tmp")

        temporary_path.write_text(
            json.dumps(
                payload,
                indent=2,
                ensure_ascii=False,
            ),
            encoding="utf-8",
        )

        temporary_path.replace(path)

    def save_channel_config(self) -> None:
        """Persist the channel selected by the server administrator."""
        path: Path = self.settings["settings_file"]
        path.parent.mkdir(parents=True, exist_ok=True)

        payload = {
            "channel_id": self.settings["channel_id"],
        }

        temporary_path = path.with_suffix(f"{path.suffix}.tmp")

        temporary_path.write_text(
            json.dumps(payload, indent=2),
            encoding="utf-8",
        )

        temporary_path.replace(path)

    async def _complete_trade(
        self,
        ctx: commands.Context[NyotaliadexBot],
        trade: dict[str, Any],
    ) -> None:
        """Swap the two characters once both sides have offered one."""
        initiator_key = str(trade["initiator"])
        recipient_key = str(trade["recipient"])

        initiator_char = trade["initiator_char"]
        recipient_char = trade["recipient_char"]

        initiator_set = self.collections.get(initiator_key, set())
        recipient_set = self.collections.get(recipient_key, set())

        # Both must still own their offered character
        if initiator_char not in initiator_set or recipient_char not in recipient_set:




            await ctx.send("One of the offered characters is no longer owned, trade cancelled, nyo~!")
            del self.pending_trades[trade["initiator"]]
            return

        # Swap
        initiator_set.discard(initiator_char)
        initiator_set.add(recipient_char)

        recipient_set.discard(recipient_char)
        recipient_set.add(initiator_char)

        self.save_collections()

        del self.pending_trades[trade["initiator"]]

        initiator_user = self.get_user(trade["initiator"])
        recipient_user = self.get_user(trade["recipient"])

        await ctx.send(
            f"Trade complete, nyo~!\n"
            f"{initiator_user.display_name if initiator_user else 'User'} "
            f"got **{recipient_char}**\n"
            f"{recipient_user.display_name if recipient_user else 'User'} "
            f"got **{initiator_char}**"
        )

    async def setup_hook(self) -> None:
        self.spawn_task = asyncio.create_task(
            self.spawn_loop(),
            name="nyotaliadex-spawn-loop",
        )

    async def close(self) -> None:
        if self.spawn_task is not None:
            self.spawn_task.cancel()

            try:
                await self.spawn_task
            except asyncio.CancelledError:
                pass

            self.spawn_task = None

        await super().close()

    async def on_ready(self) -> None:
        logger.info(
            "Logged in as %s (%s)",
            self.user,
            self.user.id if self.user else "?",
        )

    async def select_spawn_channel(
        self,
    ) -> discord.TextChannel | None:
        configured_id = self.settings["channel_id"]

        if configured_id:
            channel = self.get_channel(configured_id)

            if isinstance(channel, discord.TextChannel):
                return channel

            logger.warning(
                "SPAWN_CHANNEL_ID=%s is not an accessible text channel",
                configured_id,
            )
            return None

        channels = [
            channel
            for channel in self.get_all_channels()
            if isinstance(channel, discord.TextChannel)
        ]

        return random.choice(channels) if channels else None

    async def spawn_loop(self) -> None:
        await self.wait_until_ready()

        try:
            while not self.is_closed():
                await asyncio.sleep(self.settings["spawn_interval"])

                channel = await self.select_spawn_channel()

                if channel is None:
                    logger.warning(
                        "No accessible text channel found; "
                        "skipping this spawn"
                    )
                    continue

                async with self.state_lock:
                    if self.spawned is not None:
                        continue

                    character = random.choice(self.characters)
                    self.spawned = character
                    self.caught_by.clear()
                    self.spawn_message_channel = channel

                try:
                    await channel.send(
                        "A wild nyo appeared, quick catch it! Use `!catch`."
                    )

                    await asyncio.sleep(self.settings["catch_window"])

                except discord.DiscordException:
                    logger.exception(
                        "Could not announce or expire a spawn"
                    )

                finally:
                    async with self.state_lock:
                        if self.spawned is character:
                            await channel.send(
                                "This wild nyo ran away..."
                            )

                            self.spawned = None
                            self.caught_by.clear()
                            self.spawn_message_channel = None

        except asyncio.CancelledError:
            raise

    async def catch_character(
        self,
        user_id: int,
        guessed_name: str | None,
    ) -> tuple[str, str] | None:
        """Check a name and catch the current character if it matches."""
        async with self.state_lock:
            if self.spawned is None:
                return None

            if user_id in self.caught_by:
                return ("already_caught", "")

            character = self.spawned

            if (
                not guessed_name
                or normalize_catch_input(guessed_name.strip())
                not in catch_name_aliases(character)
            ):
                return ("wrong_guess", "")

            self.caught_by.add(user_id)

            user_key = str(user_id)

            self.collections.setdefault(
                user_key,
                set(),
            ).add(character["name"]]

            self.save_collections()

            if self.settings["first_come_first_served"]:
                self.spawned = None
                self.caught_by.clear()
                self.spawn_message_channel = None

            return ("caught", character["name"])


def register_commands(bot: NyotaliadexBot) -> None:
    @bot.event
    async def on_message(message: discord.Message) -> None:
        if message.author.bot:
            return

        content = message.content.lower()

        if "who are you" in content:
            await message.channel.send(
                "I'm Nyotaliadex, the NYO version of Italy! "
                "I catch and collect characters, nyo~!"
            )

        await bot.process_commands(message)



    @bot.command()
    async def catch(
        ctx: commands.Context[NyotaliadexBot],
        *,
        guessed_name: str | None = None,
    ) -> None:
        result = await bot.catch_character(
            ctx.author.id,
            guessed_name,
        )

        if result is None:
            await ctx.send(
                "There's nothing to catch right now, nyo~!"
            )

        elif result[0] == "already_caught":
            await ctx.send(
                "You already caught this one, nyo~!"
            )

        elif result[0] == "wrong_guess":
            await ctx.send(
                "That's not the right name, nyo~! "
            )

        else:
            await ctx.send(
                f"You've successfully caught {result[1]}!! "
                "Woah, that was quick!"
            )



    @bot.command()
    async def collection(
        ctx: commands.Context[NyotaliadexBot],
    ) -> None:
        names = sorted(
            bot.collections.get(
                str(ctx.author.id),
                set(),
            )
        )

        if not names:
            await ctx.send(
                "You haven't caught anyone yet, nyo~! Go catch some!"
            )
            return



        await ctx.send(
            f"Your collection ({len(names)}): {', '.join(names)}"
        )



    @bot.command()
    @commands.has_guild_permissions(manage_guild=True)
    async def config(
        ctx: commands.Context[NyotaliadexBot],
        channel: discord.TextChannel | None = None,
    ) -> None:
        """Show or set the channel used for automatic spawn messages."""
        if channel is None:
            configured_id = bot.settings["channel_id"]

            if not configured_id:
                await ctx.send(
                    "No spawn channel is configured. Use "
                    "`!config #channel-name`."
                )
                return

            configured_channel = bot.get_channel(configured_id)

            if isinstance(configured_channel, discord.TextChannel):
                await ctx.send(
                    "Automatic spawn messages are sent in "
                    f"{configured_channel.mention}."
                )

            else:
                await ctx.send(
                    "A spawn channel is configured, but I can no longer "
                    "access it. Set a new one with "
                    "`!config #channel-name`."
                )

            return

        permissions = channel.permissions_for(ctx.guild.me)

        if not permissions.send_messages:
            await ctx.send(
                f"I cannot send messages in {channel.mention}. "
                "Give me permission to send messages there first."
            )
            return

        bot.settings["channel_id"] = channel.id

        try:
            bot.save_channel_config()
        except OSError:
            logger.exception(
                "Could not save the configured spawn channel"
            )

            await ctx.send(
                "I couldn't save that channel configuration. "
                "Please try again."
            )
            return

        await ctx.send(
            f"Configured {channel.mention} for automatic spawn messages. "
            "I will not use other channels."
        )



    @bot.command()
    async def rarity(ctx: commands.Context[NyotaliadexBot]) -> None:
        lines = [f"{c['name']} — {c['rarity']}" for c in bot.characters]
        await ctx.send("**Character Rarities**\n" + "\n".join(lines))



    @bot.command()
    async def completion(ctx: commands.Context[NyotaliadexBot]) -> None:

        owned = bot.collections.get(str(ctx.author.id), set())
        total = len(bot.characters)
        owned_count = len(owned)
        percentage = (owned_count / total) * 100 if total else 0

        missing = [
            c["name"] for c in bot.characters
            if c["name"] not in owned
        ]

        await ctx.send(
            f"**Your Collection** ({owned_count}/{total}, {percentage:.1f}%)\n"
            f"**Owned:** {', '.join(sorted(owned)) or 'None'}\n"
            f"**Missing:** {', '.join(missing) or 'Nothing — you have them all!'}"
        )



    @bot.command()
    async def trade(
        ctx: commands.Context[NyotaliadexBot],
        partner: discord.Member,
        *,
        character: str,
    ) -> None:
        """Offer a character in a trade. The other user must accept."""
        if partner.id == ctx.author.id:
            await ctx.send("You can't trade with yourself, nyo~!")
            return

        character = canonical_character_name(character.strip())

        valid_names = {c["name"] for c in bot.characters}
        if character not in valid_names:
            await ctx.send(f"'{character}' isn't a valid character, nyo~!")
            return

        # Reject if there's already a pending trade involving either user
        for pending in bot.pending_trades.values():
            if ctx.author.id in (pending["initiator"], pending["recipient"]) or \
               partner.id in (pending["initiator"], pending["recipient"]):
                await ctx.send("One of you already has a pending trade, nyo~! Finish it first.")
                return

        giver_set = bot.collections.get(str(ctx.author.id, set())
        if character not in giver_set:

            await ctx.send(f"You don't own {character}, so you can't offer it, nyo~!")
            return

        bot.pending_trades[ctx.author.id] = {
            "initiator": ctx.author.id,
            "initiator_char": character,
            "recipient": partner.id,
            "recipient_char": None,
        }

        await ctx.send(
            f"{ctx.author.display_name} offers **{character}** to "
            f"{partner.mention}!\n"
            f"Reply with `!accept {character}` to accept and offer a character back."
        )



    @bot.command()
    async def accept(
        ctx: commands.Context[NyotaliadexBot],
        *,
        character: str,
    ) -> None:
        """Accept a pending trade and offer a character in return."""
        trade = None
        for pending in bot.pending_trades.values():
            if pending["recipient"] == ctx.author.id:
                trade = pending
                break

        if trade is None:
            await ctx.send("You have no pending trade to accept, nyo~!")
            return

        character = canonical_character_name(character.strip())

        valid_names = {c["name"] for c in bot.characters}
        if character not in valid_names:
            await ctx.send(f"'{character}' isn't a valid character, nyo~!")
            return

        receiver_set = bot.collections.get(str(ctx.author.id, set())
        if character not in receiver_set:

            await ctx.send(f"You don't own {character}, so you can't offer it, nyo~!")
            return

        trade["recipient_char"] = character

        await bot._complete_trade(ctx, trade)



    @bot.command()
    async def decline(ctx: commands.Context[NyotaliadexBot]) -> None:


        for user_id, pending in list(bot.pending_trades.items()):
            if ctx.author.id in (pending["initiator"], pending["recipient"]):
                del bot.pending_trades[user_id]
                await ctx.send("Trade cancelled, nyo~!")
                return

        await ctx.send("You have no pending trade to decline, nyo~!")



    @bot.event
    async def on_command_error(
        ctx: commands.Context[NyotaliadexBot],
        error: commands.CommandError,
    ) -> None:
        if isinstance(error, commands.MissingPermissions):
            await ctx.send(
                "Only a server administrator with Manage Server permission "
                "can use `!config`."
            )

        elif isinstance(error, commands.BadArgument):
            await ctx.send(
                "I couldn't find that text channel. Try mentioning it, "
                "like `!config #nyo-catches`."
            )

        elif isinstance(error, commands.MissingRequiredArgument):
            await ctx.send(
                "Please provide a text channel, like "
                "`!config #nyo-catches`."
            )

        elif not isinstance(error, commands.CommandNotFound):
            logger.exception(
                "Unhandled command error",
                exc_info=error,
            )



def main() -> None:
    settings = load_settings()
    bot = NyotaliadexBot(settings)
    register_commands(bot)
    bot.run(settings["token"], log_handler=None)


if __name__ == "__main__":
    main()
