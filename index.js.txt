const { 
  Client, 
  GatewayIntentBits, 
  SlashCommandBuilder, 
  REST, 
  Routes 
} = require("discord.js");
const fs = require("fs");
const crypto = require("crypto");
const config = require("./config.json");

const client = new Client({
  intents: [
    GatewayIntentBits.Guilds,
    GatewayIntentBits.GuildMessages,
    GatewayIntentBits.MessageContent
  ]
});

const TOKEN = process.env.TOKEN;
const dataFile = "./ggs.json";

// ===== FUNÇÕES =====
function loadData() {
  if (!fs.existsSync(dataFile)) return {};
  return JSON.parse(fs.readFileSync(dataFile));
}

function saveData(data) {
  fs.writeFileSync(dataFile, JSON.stringify(data, null, 2));
}

function hashImage(url) {
  return crypto.createHash("md5").update(url).digest("hex");
}

// ===== SLASH COMMANDS =====
const commands = [
  new SlashCommandBuilder()
    .setName("ggrank")
    .setDescription("Mostra o ranking de GG do canal"),

  new SlashCommandBuilder()
    .setName("ggreset")
    .setDescription("Reseta o contador de GG deste canal")
];

const rest = new REST({ version: "10" }).setToken(TOKEN);

client.once("ready", async () => {
  console.log(`✅ Bot online como ${client.user.tag}`);

  await rest.put(
    Routes.applicationCommands(client.user.id),
    { body: commands.map(c => c.toJSON()) }
  );

  console.log("✅ Slash commands registrados");
});

// ===== CONTADOR =====
client.on("messageCreate", message => {
  if (message.author.bot) return;
  if (!message.content.includes(config.keyword)) return;

  const channelId = message.channel.id;
  const userId = message.author.id;
  const data = loadData();

  if (!data[channelId]) {
    data[channelId] = { users: {}, images: [] };
  }

  // Verificar imagens repetidas
  if (message.attachments.size > 0) {
    for (const attachment of message.attachments.values()) {
      const imgHash = hashImage(attachment.url);
      if (data[channelId].images.includes(imgHash)) return;
      data[channelId].images.push(imgHash);
    }
  }

  data[channelId].users[userId] = (data[channelId].users[userId] || 0) + 1;
  saveData(data);
});

// ===== SLASH INTERACTIONS =====
client.on("interactionCreate", interaction => {
  if (!interaction.isChatInputCommand()) return;

  const data = loadData();
  const channelId = interaction.channel.id;

  if (interaction.commandName === "ggreset") {
    data[channelId] = { users: {}, images: [] };
    saveData(data);
    return interaction.reply("✅ Contador de GG resetado neste canal.");
  }

  if (interaction.commandName === "ggrank") {
    if (!data[channelId] || Object.keys(data[channelId].users).length === 0) {
      return interaction.reply("📭 Nenhum GG registrado neste canal.");
    }

    const ranking = Object.entries(data[channelId].users)
      .sort((a, b) => b[1] - a[1])
      .map(([id, count], i) => `${i + 1}️⃣ <@${id}> — **${count}**`)
      .join("\n");

    interaction.reply(`🏆 **Ranking GG do Canal**\n\n${ranking}`);
  }
});

client.login(TOKEN);
