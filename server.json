const express = require('express');
const cors = require('cors');
const axios = require('axios');

const app = express();
app.use(cors());
app.use(express.json());

// Configurações Evolution API
const EVOLUTION_API_URL = process.env.EVOLUTION_API_URL || 'https://sua-evolution-api.up.railway.app';
const EVOLUTION_API_KEY = process.env.EVOLUTION_API_KEY || 'SuaChaveSuperSecreta123!';

// Armazenamento temporário (em produção usar PostgreSQL)
let messages = {};
let leads = {};

// ===== HEALTH CHECK =====
app.get('/', (req, res) => {
  res.json({ 
    status: 'ok', 
    message: 'RevisionWhats Backend API',
    version: '1.0.0'
  });
});

// ===== CONECTAR WHATSAPP =====
app.post('/api/whatsapp/connect', async (req, res) => {
  const { userId, userName } = req.body;
  const instanceName = `consultor-${userId}`;

  try {
    console.log(`🔌 Criando instância: ${instanceName}`);
    
    const response = await axios.post(
      `${EVOLUTION_API_URL}/instance/create`,
      {
        instanceName: instanceName,
        qrcode: true,
        integration: 'WHATSAPP-BAILEYS'
      },
      {
        headers: {
          'Content-Type': 'application/json',
          'apikey': EVOLUTION_API_KEY
        }
      }
    );

    console.log('✅ Instância criada:', response.data);

    res.json({
      success: true,
      instanceName: instanceName,
      qrcode: response.data.qrcode?.code || response.data.qrcode,
      message: 'Escaneie o QR Code no seu WhatsApp'
    });

  } catch (error) {
    console.error('❌ Erro ao conectar:', error.response?.data || error.message);
    res.status(500).json({ 
      success: false,
      error: error.response?.data || error.message 
    });
  }
});

// ===== STATUS CONEXÃO =====
app.get('/api/whatsapp/status/:userId', async (req, res) => {
  const instanceName = `consultor-${req.params.userId}`;

  try {
    const response = await axios.get(
      `${EVOLUTION_API_URL}/instance/connectionState/${instanceName}`,
      {
        headers: { 'apikey': EVOLUTION_API_KEY }
      }
    );

    res.json({
      connected: response.data.state === 'open',
      status: response.data.state,
      instanceName: instanceName
    });

  } catch (error) {
    res.json({ 
      connected: false, 
      status: 'disconnected',
      error: error.message
    });
  }
});

// ===== ENVIAR MENSAGEM =====
app.post('/api/messages/send', async (req, res) => {
  const { userId, phone, message, leadId } = req.body;
  const instanceName = `consultor-${userId}`;

  try {
    console.log(`📤 Enviando mensagem para ${phone}`);

    // Limpar número (remover +, espaços, hífen)
    const cleanPhone = phone.replace(/[^0-9]/g, '');

    const response = await axios.post(
      `${EVOLUTION_API_URL}/message/sendText/${instanceName}`,
      {
        number: cleanPhone,
        text: message
      },
      {
        headers: {
          'Content-Type': 'application/json',
          'apikey': EVOLUTION_API_KEY
        }
      }
    );

    console.log('✅ Mensagem enviada:', response.data);

    // Salvar no histórico (memória temporária)
    if (!messages[leadId]) messages[leadId] = [];
    messages[leadId].push({
      type: 'sent',
      text: message,
      time: new Date().toLocaleTimeString('pt-BR', { hour: '2-digit', minute: '2-digit' }),
      status: 'sent',
      timestamp: Date.now()
    });

    res.json({ 
      success: true, 
      data: response.data,
      messageId: response.data.key?.id
    });

  } catch (error) {
    console.error('❌ Erro ao enviar:', error.response?.data || error.message);
    res.status(500).json({ 
      success: false,
      error: error.response?.data || error.message 
    });
  }
});

// ===== LISTAR MENSAGENS =====
app.get('/api/messages/:leadId', (req, res) => {
  const { leadId } = req.params;
  const leadMessages = messages[leadId] || [];

  res.json({
    success: true,
    messages: leadMessages
  });
});

// ===== WEBHOOK (Receber Mensagens) =====
app.post('/webhook/whatsapp', (req, res) => {
  const { event, data } = req.body;

  console.log('📨 Webhook recebido:', event);

  if (event === 'messages.upsert') {
    const message = data;
    
    // Se não é mensagem nossa (fromMe = false)
    if (!message.key?.fromMe) {
      const phone = message.key?.remoteJid?.replace('@s.whatsapp.net', '');
      const text = message.message?.conversation || 
                   message.message?.extendedTextMessage?.text || 
                   '[Mídia]';

      console.log(`📥 Mensagem recebida de ${phone}: ${text}`);

      // Salvar no histórico (usar phone como leadId temporário)
      const leadId = phone;
      if (!messages[leadId]) messages[leadId] = [];
      
      messages[leadId].push({
        type: 'received',
        text: text,
        time: new Date().toLocaleTimeString('pt-BR', { hour: '2-digit', minute: '2-digit' }),
        status: 'delivered',
        timestamp: Date.now()
      });

      // TODO: Notificar frontend via WebSocket ou SSE
    }
  }

  res.sendStatus(200);
});

// ===== ADICIONAR LEAD =====
app.post('/api/leads', (req, res) => {
  const { userId, name, phone } = req.body;
  const leadId = Date.now().toString();

  leads[leadId] = {
    id: leadId,
    consultorId: userId,
    name: name,
    phone: phone,
    createdAt: new Date().toISOString()
  };

  res.json({
    success: true,
    lead: leads[leadId]
  });
});

// ===== LISTAR LEADS =====
app.get('/api/leads/:userId', (req, res) => {
  const { userId } = req.params;
  const userLeads = Object.values(leads).filter(l => l.consultorId === userId);

  res.json({
    success: true,
    leads: userLeads
  });
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`🚀 Backend rodando na porta ${PORT}`);
  console.log(`📡 Evolution API: ${EVOLUTION_API_URL}`);
});
