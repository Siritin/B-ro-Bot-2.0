// Discord.js v14 Bot: 2 Systeme parallel, unterschiedliche Kategorien

const { Client, GatewayIntentBits, EmbedBuilder, ActionRowBuilder, ButtonBuilder, ButtonStyle, Events, ChannelType } = require('discord.js');

const client = new Client({
  intents: [GatewayIntentBits.Guilds, GatewayIntentBits.GuildVoiceStates]
});

// ---------------------------
// TOKEN (auf Railway als ENV setzen)
const TOKEN = process.env.TOKEN;

// ---------------------------
// System 1
const CHANNEL_1_ID_1 = "1483918907747340318"; // Voice Wartezimmer 1
const CHANNEL_2_ID_1 = "1484291035751514122"; // Text Channel 1
const CATEGORY_1_ID = "1477718723203961124"; // Kategorie 1

// System 2
const CHANNEL_1_ID_2 = "1469990647732895785"; // Voice Wartezimmer 2
const CHANNEL_2_ID_2 = "1484300167820607688"; // Text Channel 2
const CATEGORY_2_ID = "1469809618430459964"; // Kategorie 2

// ---------------------------
// Map für wartende User pro System
let waitingUsers = new Map();
let messageRefs = new Map();

// ---------------------------
// VoiceStateUpdate: User joint oder verlässt Voice Channel
client.on(Events.VoiceStateUpdate, async (oldState, newState) => {

  // --- System 1 ---
  if (newState.channelId === CHANNEL_1_ID_1 && oldState.channelId !== CHANNEL_1_ID_1) {
    waitingUsers.set('1', newState.member);
    const embed = new EmbedBuilder().setTitle("Neuer User wartet (System 1)").setDescription(`${newState.member.user.tag} wartet im Channel`).setColor(0x00ff00);
    const button = new ButtonBuilder().setCustomId("claim_user_1").setLabel("Übernehmen").setStyle(ButtonStyle.Primary);
    const row = new ActionRowBuilder().addComponents(button);
    const channel = await client.channels.fetch(CHANNEL_2_ID_1);
    const msg = await channel.send({ embeds: [embed], components: [row] });
    messageRefs.set('1', msg);
  }

  if (oldState.channelId === CHANNEL_1_ID_1 && newState.channelId !== CHANNEL_1_ID_1) {
    if (messageRefs.has('1')) {
      const disabledRow = new ActionRowBuilder().addComponents(new ButtonBuilder().setCustomId("claim_user_1").setLabel("Übernehmen").setStyle(ButtonStyle.Secondary).setDisabled(true));
      await messageRefs.get('1').edit({ components: [disabledRow] });
    }
    waitingUsers.delete('1');
  }

  // --- System 2 ---
  if (newState.channelId === CHANNEL_1_ID_2 && oldState.channelId !== CHANNEL_1_ID_2) {
    waitingUsers.set('2', newState.member);
    const embed = new EmbedBuilder().setTitle("Neuer User wartet (System 2)").setDescription(`${newState.member.user.tag} wartet im Channel`).setColor(0x00ff00);
    const button = new ButtonBuilder().setCustomId("claim_user_2").setLabel("Übernehmen").setStyle(ButtonStyle.Primary);
    const row = new ActionRowBuilder().addComponents(button);
    const channel = await client.channels.fetch(CHANNEL_2_ID_2);
    const msg = await channel.send({ embeds: [embed], components: [row] });
    messageRefs.set('2', msg);
  }

  if (oldState.channelId === CHANNEL_1_ID_2 && newState.channelId !== CHANNEL_1_ID_2) {
    if (messageRefs.has('2')) {
      const disabledRow = new ActionRowBuilder().addComponents(new ButtonBuilder().setCustomId("claim_user_2").setLabel("Übernehmen").setStyle(ButtonStyle.Secondary).setDisabled(true));
      await messageRefs.get('2').edit({ components: [disabledRow] });
    }
    waitingUsers.delete('2');
  }
});

// ---------------------------
// Button Interaction
client.on(Events.InteractionCreate, async interaction => {
  if (!interaction.isButton()) return;

  let systemKey;
  if (interaction.customId === "claim_user_1" || interaction.customId === "close_user_1") systemKey = '1';
  if (interaction.customId === "claim_user_2" || interaction.customId === "close_user_2") systemKey = '2';

  if (!systemKey) return;
  const waitingUser = waitingUsers.get(systemKey);
  if (!waitingUser) return interaction.reply({ content: "Kein User vorhanden", ephemeral: true });

  let voiceChannel = interaction.member.voice.channel;
  const allowedCategory = systemKey === '1' ? CATEGORY_1_ID : CATEGORY_2_ID;

  if (interaction.customId.startsWith('claim_user')) {
    if (!voiceChannel || voiceChannel.parentId !== allowedCategory) {
      return interaction.reply({ content: "Du darfst nur Channels aus der erlaubten Kategorie verwenden!", ephemeral: true });
    }

    await waitingUser.voice.setChannel(voiceChannel);

    const embed = new EmbedBuilder().setTitle(`User übernommen (System ${systemKey})`).setDescription(`${waitingUser.user.tag} ist jetzt bei ${interaction.member.user.tag}`).setColor(0x0099ff);
    const closeButton = new ButtonBuilder().setCustomId(`close_user_${systemKey}`).setLabel("Schließen").setStyle(ButtonStyle.Danger);
    const row = new ActionRowBuilder().addComponents(closeButton);

    await interaction.update({ embeds: [embed], components: [row] });

  } else if (interaction.customId.startsWith('close_user')) {
    await waitingUser.voice.disconnect();
    const embed = new EmbedBuilder().setTitle(`Session geschlossen (System ${systemKey})`).setDescription("User wurde gekickt").setColor(0xff0000);
    const disabledRow = new ActionRowBuilder().addComponents(new ButtonBuilder().setCustomId("closed").setLabel("Geschlossen").setStyle(ButtonStyle.Secondary).setDisabled(true));

    waitingUsers.delete(systemKey);
    await interaction.update({ embeds: [embed], components: [disabledRow] });
  }
});

// ---------------------------
client.login(TOKEN);
