# Discord Economy Bot Tutorial

A beginner-friendly Discord bot that demonstrates the fundamentals of building an economy system with persistent storage. This bot serves as a foundation for more complex economy bots with features like shops, gambling, trading, and more.

## What This Bot Does

This bot implements a basic economy system where users can:
- **Work** to earn coins
- **Check Balance** to view wallet and bank amounts
- **Deposit** coins from wallet to bank
- **Withdraw** coins from bank to wallet
- **Send** coins to other users

Each user has two storage locations:
- **Wallet**: Immediate spending money (used for transactions, sending, and at risk in games/robberies in advanced bots)
- **Bank**: Safe storage for long-term savings (player-controlled savings account)

## Prerequisites

Before running this bot, you need:
- Python 3.8 or higher
- A Discord Bot Token ([Get one here](https://discord.com/developers/applications))
- Basic understanding of Python async/await

## Installation

1. **Clone or download this repository**

2. **Install required packages:**
```bash
pip install discord.py python-dotenv sqlalchemy aiosqlite
```

3. **Create a `.env` file in the project root:**
```env
BOT_TOKEN=your_bot_token_here
DATABASE_URL=sqlite+aiosqlite:///data/economy.db
```

**Note:** The bot will automatically create a `data/` directory for the database file if it doesn't exist. You can also use `DB_URL` or `Db_URL` as alternative environment variable names.

4. **Run the bot:**
```bash
python bot.py
```

## How It Works

### 1. Environment Setup
```python
load_dotenv()
_token = os.getenv("BOT_TOKEN")
if not _token:
    raise ValueError("BOT_TOKEN not found in environment variables")
TOKEN: str = _token

# Support multiple environment variable names for database URL
_db_env = os.getenv("DATABASE_URL") or os.getenv("DB_URL") or os.getenv("Db_URL")
default_db = "sqlite+aiosqlite:///data/economy.db"
DB_URL = _db_env if _db_env else default_db
```

**What this does:**
- Loads environment variables from `.env` file
- Validates that the bot token exists before starting
- Supports multiple database URL variable names (`DATABASE_URL`, `DB_URL`, `Db_URL`)
- Defaults to `data/economy.db` if no database URL is specified
- Strips surrounding quotes from environment variables if present
- **Automatically creates the `data/` directory** if it doesn't exist

This keeps sensitive data separate from your code and provides flexible configuration options.

### 2. Database Model (SQLAlchemy ORM)
```python
class EconUser(Base):
    __tablename__ = "econ_users"

    user_id: Mapped[int] = mapped_column(BigInteger, primary_key=True)
    wallet: Mapped[int] = mapped_column(Integer, default=0)
    bank: Mapped[int] = mapped_column(Integer, default=0)
```

**What this does:**
- Creates a database table called `econ_users`
- Each user has a unique `user_id` (their Discord ID)
- Tracks `wallet` and `bank` balances (both default to 0)
- Uses SQLAlchemy ORM to interact with the database without writing raw SQL

**Why BigInteger for user_id?**
Discord IDs are 64-bit integers that exceed the range of standard 32-bit integers.

### 3. Database Manager

The `EconManager` class handles all database operations:

#### `get_user(user_id)` - Retrieve or Create User
```python
async def get_user(self, user_id: int) -> EconUser:
    result = await self.session.execute(select(EconUser).where(EconUser.user_id == user_id))
    user = result.scalar_one_or_none()
    if not user:
        user = EconUser(user_id=user_id, wallet=0, bank=0)
        self.session.add(user)
        await self.session.commit()
    return user
```

**Flow:**
1. Query database for user
2. If not found, create new user with 0 balance
3. Save to database
4. Return user object

#### `add_coins(user_id, amount)` - Add Money to Wallet
```python
async def add_coins(self, user_id: int, amount: int) -> EconUser:
    user = await self.get_user(user_id)
    user.wallet += amount
    await self.session.commit()
    return user
```

**Flow:**
1. Get user from database
2. Increase wallet by amount
3. Save changes
4. Return updated user

#### `transfer(user_id, amount, from_wallet)` - Move Money
```python
async def transfer(self, user_id: int, amount: int, from_wallet: bool) -> bool:
    user = await self.get_user(user_id)
    if from_wallet:
        if user.wallet >= amount:
            user.wallet -= amount
            user.bank += amount
        else:
            return False
    else:
        if user.bank >= amount:
            user.bank -= amount
            user.wallet += amount
        else:
            return False
    await self.session.commit()
    return True
```

**Flow:**
1. Get user from database
2. Check if they have enough money
3. If yes: move money between wallet/bank
4. If no: return False (transaction failed)
5. Save changes and return success status

### 4. Slash Commands

Discord slash commands are modern, type-safe commands that appear in Discord's command picker.

#### `/balance` - Check Your Money
```python
@bot.tree.command(name="balance", description="Check your wallet and bank balance.")
async def balance(interaction: discord.Interaction):
    async with AsyncSessionLocal() as session:
        manager = EconManager(session)
        user = await manager.get_user(interaction.user.id)

        embed = discord.Embed(
            title=f"{interaction.user.name}'s Balance",
            description=f"💰 Wallet: **{user.wallet}**\n🏦 Bank: **{user.bank}**",
            color=discord.Color.gold(),
        )
        await interaction.response.send_message(embed=embed)
```

**Breakdown:**
1. `@bot.tree.command` - Registers a slash command
2. `async with AsyncSessionLocal()` - Opens database connection
3. `manager.get_user()` - Fetches user data
4. Creates pretty embed with balance info
5. Sends response to Discord

#### `/work` - Earn Coins
```python
@bot.tree.command(name="work", description="Work to earn some coins.")
async def work(interaction: discord.Interaction):
    amount = random.randint(50, 200)
    async with AsyncSessionLocal() as session:
        manager = EconManager(session)
        await manager.add_coins(interaction.user.id, amount)

    await interaction.response.send_message(f"💼 You worked hard and earned **{amount}** coins!")
```

**Breakdown:**
1. Generate random amount (50-200 coins)
2. Add coins to user's wallet
3. Notify user of earnings

#### `/deposit <amount>` - Store Money Safely
```python
@bot.tree.command(name="deposit", description="Deposit coins into your bank.")
@app_commands.describe(amount="Amount of coins to deposit.")
async def deposit(interaction: discord.Interaction, amount: int):
    async with AsyncSessionLocal() as session:
        manager = EconManager(session)
        success = await manager.transfer(interaction.user.id, amount, from_wallet=True)

    if success:
        await interaction.response.send_message(f"🏦 You deposited **{amount}** coins into your bank.")
    else:
        await interaction.response.send_message("❌ Not enough coins in your wallet.")
```

**Breakdown:**
1. `@app_commands.describe` - Adds parameter description in Discord
2. Attempts to transfer from wallet to bank
3. Sends success or failure message

#### `/withdraw <amount>` - Get Money from Bank

Similar to deposit, but moves money from bank to wallet.

#### `/send <user> <amount>` - Send Coins to Another User
```python
@bot.tree.command(name="send", description="Send coins to another user.")
@app_commands.describe(user="The user to send coins to.", amount="Amount of coins to send.")
async def send(interaction: discord.Interaction, user: discord.User, amount: int):
    # Validation checks
    if amount <= 0:
        await interaction.response.send_message("❌ You must send a positive amount of coins.")
        return
    
    if user.id == interaction.user.id:
        await interaction.response.send_message("❌ You can't send coins to yourself!")
        return
    
    if user.bot:
        await interaction.response.send_message("❌ You can't send coins to bots!")
        return

    async with AsyncSessionLocal() as session:
        manager = EconManager(session)
        
        # Get sender
        sender = await manager.get_user(interaction.user.id)
        
        # Check if sender has enough money
        if sender.wallet < amount:
            await interaction.response.send_message(f"❌ You don't have enough coins!")
            return
        
        # Get or create recipient
        recipient = await manager.get_user(user.id)
        
        # Transfer coins
        sender.wallet -= amount
        recipient.wallet += amount
        await session.commit()

    await interaction.response.send_message(
        f"💸 You sent **{amount}** coins to {user.mention}!\n"
        f"Your new balance: **{sender.wallet}** coins"
    )
```

**Breakdown:**
1. Validates the amount is positive
2. Prevents sending to yourself or bots
3. Checks sender has enough coins in wallet
4. Transfers coins from sender's wallet to recipient's wallet
5. Sends confirmation message with updated balance

**Important Notes:**
- Coins are sent from **wallet to wallet** (not bank)
- Recipient doesn't need to have used the bot before (account created automatically)
- Transaction is atomic - either fully succeeds or fully fails

### 5. Async/Await Pattern

This bot uses async programming for efficiency:
```python
async def main():
    await init_db()  # Create database tables
    async with bot:
        await bot.start(TOKEN)  # Start bot
```

**Why async?**
- Handles multiple users simultaneously
- Doesn't block while waiting for database/Discord
- Required by discord.py 2.0+

### 6. Database Session Management
```python
async with AsyncSessionLocal() as session:
    manager = EconManager(session)
    # Do database operations
```

**The `async with` pattern:**
- Opens a database session
- Automatically commits changes on success
- Automatically rolls back on errors
- Closes session when done
- Prevents database locks and connection leaks

### 7. Understanding the Economy Flow

**How Money Moves:**

1. **Earning** (`/work`) → Adds coins to your **wallet**
2. **Depositing** (`/deposit 100`) → Moves coins from **wallet** to **bank** (for safekeeping)
3. **Withdrawing** (`/withdraw 50`) → Moves coins from **bank** to **wallet** (to spend)
4. **Sending** (`/send @user 25`) → Transfers coins from your **wallet** to another user's **wallet**

**Why Two Accounts?**
- **Wallet**: Active spending money, used for all transactions
- **Bank**: Manual savings account you control, protected in advanced bots from robbery/gambling losses

**Global vs Per-Server:**
- Currently, the economy is **global** - your balance follows you across all servers the bot is in
- To make it per-server, you'd need to add `guild_id` to the database model (see expansion ideas below)

## Expanding This Bot

This foundation can be extended with:

### Economy Features
- **Shops**: Buy items with virtual currency
- **Inventory System**: Store and use items
- **Job System**: Different jobs with different payouts
- **Daily Rewards**: Login bonuses
- **Cooldowns**: Prevent spam (using timestamps)

### Risk/Reward Features
- **Gambling**: Coinflip, slots, blackjack
- **Crime**: High risk, high reward actions
- **Robbery**: Steal from other users' wallets (not banks!)

### Social Features
- **Trading**: Exchange items/money with other users (already have basic `/send` command!)
- **Leaderboards**: Top richest users
- **Gifting**: Send money to friends (already implemented with `/send`)
- **Multipliers**: Boost earnings based on achievements
- **Request/Pay System**: Users can request money from others

### Advanced Features
- **Prestige System**: Reset progress for permanent bonuses
- **Stock Market**: Buy/sell virtual stocks
- **Business Ownership**: Passive income generation
- **Achievements**: Unlock rewards for milestones

## Common Issues & Solutions

### "Object has no attribute '__aenter__'"
**Problem:** Using wrong sessionmaker  
**Solution:** Use `async_sessionmaker` not `sessionmaker`

### "Cannot assign to attribute 'wallet'"
**Problem:** Using old SQLAlchemy Column syntax  
**Solution:** Use `Mapped[int]` and `mapped_column()`

### Bot doesn't respond to commands
**Problem:** Commands not synced  
**Solution:** Wait a few minutes after starting bot, or use `await bot.tree.sync()` in `on_ready`

### "TOKEN is None"
**Problem:** `.env` file not loaded or BOT_TOKEN missing  
**Solution:** Check `.env` exists and contains valid token

## Best Practices Demonstrated

1. **Separation of Concerns**: Database logic in `EconManager`, commands separate
2. **Error Handling**: Check balances before transactions
3. **Type Safety**: Type hints for better IDE support and fewer bugs
4. **Environment Variables**: Keep secrets out of code
5. **Async Best Practices**: Proper session management with context managers
6. **User Feedback**: Clear success/error messages with emojis

## File Structure for Larger Projects

As your bot grows, consider organizing like this:
```
economy-bot/
├── bot.py              # Main bot file
├── data/               # Database files (auto-created)
│   └── economy.db      # SQLite database
├── cogs/
│   ├── economy.py      # Economy commands
│   ├── gambling.py     # Gambling commands
│   └── shop.py         # Shop commands
├── database/
│   ├── models.py       # Database models
│   └── manager.py      # Database operations
├── utils/
│   ├── checks.py       # Custom checks (cooldowns, etc.)
│   └── embeds.py       # Reusable embed templates
├── .env                # Environment variables
├── .gitignore          # Ignore data/, .env, etc.
├── requirements.txt    # Python dependencies
└── README.md           # Documentation
```

**Important:** Add `data/` and `.env` to your `.gitignore` to avoid committing sensitive data or database files to version control.

## Next Steps

1. **Add Cooldowns**: Prevent users from spamming `/work` (use timestamps in database)
2. **Per-Server Economy**: Add `guild_id` to make economies separate per server
3. **Create a Shop**: Let users buy items
4. **Add Inventory**: Track purchased items
5. **Implement Daily Command**: Give daily login rewards
6. **Add Leaderboard**: Show top users by balance (global or per-server)
7. **Create Games**: Coinflip, slots, or blackjack
8. **Transaction History**: Log all sends/receives for transparency

## Resources

- [discord.py Documentation](https://discordpy.readthedocs.io/)
- [SQLAlchemy Async Documentation](https://docs.sqlalchemy.org/en/20/orm/extensions/asyncio.html)
- [Discord Developer Portal](https://discord.com/developers/docs)

## License

This is a tutorial project - feel free to use, modify, and expand upon it for your own Discord bots!

---

**Happy Bot Building! 🤖💰**