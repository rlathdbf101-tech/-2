import discord
from discord import app_commands
from datetime import datetime
import json
import os

class MyClient(discord.Client):
    def __init__(self):
        intents = discord.Intents.default()
        intents.message_content = True
        intents.members = True
        super().__init__(intents=intents)
        self.tree = app_commands.CommandTree(self)
        
    async def setup_hook(self):
        await self.tree.sync()

client = MyClient()

DB_FILE = "attendance_data.json"

def load_data():
    if os.path.exists(DB_FILE):
        with open(DB_FILE, 'r', encoding='utf-8') as f:
            return json.load(f)
    return {}

def save_data(data):
    serializable_data = {}
    for user_id, info in data.items():
        temp_info = info.copy()
        if temp_info["last_attendance"] and not isinstance(temp_info["last_attendance"], str):
            temp_info["last_attendance"] = temp_info["last_attendance"].isoformat()
        serializable_data[str(user_id)] = temp_info
        
    with open(DB_FILE, 'w', encoding='utf-8') as f:
        json.dump(serializable_data, f, indent=4)

attendance_db = load_data()

ALLOWED_CHANNEL_ID = 1507313960712536194

@client.event
async def on_ready():
    try:
        synced = await client.tree.sync()
        print(f"Synced {len(synced)} commands.")
    except Exception as e:
        print(f"Error: {e}")
    print(f"Logged in as {client.user}")

@client.event
async def on_member_join(member):
    welcome_channel_id = 1506199792055881760
    channel = client.get_channel(welcome_channel_id)
    if channel:
        await channel.send(f'{member.mention}님, 서버에 오신 것을 환영합니다!')

@client.tree.command(name="출석", description="출석체크 하기")
async def attendance(interaction: discord.Interaction):
    if interaction.channel_id != ALLOWED_CHANNEL_ID:
        await interaction.response.send_message(
            f"❌ 출석 체크는 <#{ALLOWED_CHANNEL_ID}> 채널에서만 이용할 수 있습니다.",
            ephemeral=True
        )
        return

    user_id = str(interaction.user.id)
    now = datetime.now()

    joined_at = interaction.user.joined_at.replace(tzinfo=None)
    days_since_join = (now - joined_at).days + 1

    if user_id not in attendance_db:
        attendance_db[user_id] = {"total_days": 0, "streak_days": 0, "last_attendance": None}

    user_data = attendance_db[user_id]
    
    if user_data["last_attendance"] and isinstance(user_data["last_attendance"], str):
        user_data["last_attendance"] = datetime.fromisoformat(user_data["last_attendance"])

    if user_data["last_attendance"]:
        last_time = user_data["last_attendance"]
        time_passed = now - last_time
        hours_passed = time_passed.total_seconds() / 3600

        if hours_passed < 24:
            remaining_seconds = 86400 - time_passed.total_seconds()
            hours = int(remaining_seconds // 3600)
            minutes = int((remaining_seconds % 3600) // 60)
            await interaction.response.send_message(
                f"❌ **쿨타임 중입니다!** 다음 출석까지 **{hours}시간 {minutes}분** 남았습니다.", 
                ephemeral=True
            )
            return

        if 24 <= hours_passed <= 48:
            user_data["streak_days"] += 1
        else:
            user_data["streak_days"] = 1
    else:
        user_data["streak_days"] = 1

    user_data["total_days"] += 1
    user_data["last_attendance"] = now
    attendance_db[user_id] = user_data

    save_data(attendance_db)

    embed = discord.Embed(title="출석 체크 완료", color=discord.Color.green())
    embed.add_field(name="서버 가입 기간", value=f"가입한 지 **{days_since_join}일** 차", inline=False)
    embed.add_field(name="연속 출석일", value=f"**{user_data['streak_days']}일** 연속", inline=True)
    embed.add_field(name="누적 출석일", value=f"총 **{user_data['total_days']}일**", inline=True)
    
    await interaction.response.send_message(embed=embed)
    
from datetime import datetime, timezone

@client.tree.command(name="출석조회", description="[관리자 전용] 특정 유저의 출석 정보를 비밀리에 조회합니다.")
@app_commands.describe(유저="조회할 멤버 선택")
@app_commands.checks.has_any_role('교주', '부교주',) 
async def check_user_status(interaction: discord.Interaction, 유저: discord.Member):
    
    attendance_db = load_data() 
    data = attendance_db.get(str(유저.id), {})
    
    total_days = data.get("total_days", 0)
    streak_days = data.get("streak_days", 0)
    
    days_since_joined = (datetime.now(timezone.utc) - 유저.joined_at).days if 유저.joined_at else "알 수 없음"

    embed = discord.Embed(title=f"🔍 {유저.display_name}님의 정보", color=0x3498db)
    embed.add_field(name="서버 가입", value=f"들어온 지 **{days_since_joined}일**째", inline=False)
    embed.add_field(name="총 출석", value=f"**{total_days}회**", inline=True)
    embed.add_field(name="연속 출석", value=f"**{streak_days}일**", inline=True)
    
    await interaction.response.send_message(embed=embed, ephemeral=True)

@check_user_status.error
async def check_user_status_error(interaction: discord.Interaction, error: app_commands.AppCommandError):
    if isinstance(error, app_commands.errors.MissingAnyRole):
        await interaction.response.send_message("❌ 권한이 없습니다.", ephemeral=True)

client.run('MTUwNjI2NjQ4ODgyNTE4ODU0NA.Gp17I4.Z92dnPGHTzlqvnpzZjLqvu9-MamiPIf2xhDxxs')
