
1. Arquivo .env

OPENAI_API_KEY=your_openai_api_key
DB_URI=mongodb://localhost:27017/seu_banco

2. Configuração do i18next (i18n.js)

// config/i18n.js
const i18next = require('i18next');
const Backend = require('i18next-fs-backend');
const LanguageDetector = require('i18next-http-middleware').LanguageDetector;

i18next
  .use(Backend)
  .use(LanguageDetector)
  .init({
    fallbackLng: 'pt',
    preload: ['en', 'pt'],
    backend: {
      loadPath: './locales/{{lng}}/{{ns}}.json',
    },
    detection: {
      order: ['cookie', 'header'],
      caches: ['cookie'],
    },
  });

module.exports = i18next;

3. Servidor Express (app.js)

// app.js
const express = require('express');
const helmet = require('helmet');
const csrf = require('csurf');
const rateLimit = require('express-rate-limit');
const i18nextMiddleware = require('i18next-http-middleware');
const i18n = require('./config/i18n');
const perguntaRoutes = require('./routes/perguntaRoutes');
const AppError = require('./utils/AppError');
const AiError = require('./utils/AiError');
const path = require('path');
const cookieParser = require('cookie-parser');
require('dotenv').config();

const app = express();

// Middleware
app.use(express.json());
app.use(helmet());
app.use(cookieParser());
app.use(i18nextMiddleware.handle(i18n));

// Serve arquivos estáticos da pasta 'public'
app.use(express.static(path.join(__dirname, 'public')));

// Configuração de segurança
const csrfProtection = csrf({ cookie: true });
const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutos
  max: 100, // limita cada IP a 100 requisições por janela
});

app.use(csrfProtection);
app.use(limiter);

// Rotas
app.use('/pergunta', perguntaRoutes);

app.get('/', (req, res) => {
  res.sendFile(path.join(__dirname, 'public', 'index.html'), { csrfToken: req.csrfToken() });
});

// Tratamento de erros
app.use((err, req, res, next) => {
  if (err instanceof AppError) {
    return res.status(err.statusCode).json({ status: 'erro', mensagem: err.message });
  }
  if (err instanceof AiError) {
    return res.status(err.statusCode).json({ status: 'erro', mensagem: err.message });
  }
  console.error(err);
  res.status(500).json({ status: 'erro', mensagem: 'Erro interno no servidor.' });
});

module.exports = app;

4. Rota de Pergunta (perguntaRoutes.js)

// routes/perguntaRoutes.js
const express = require('express');
const { body } = require('express-validator');
const validateWithYup = require('../middlewares/validateWithYup');
const handleValidationErrors = require('../middlewares/handleValidationErrors');
const perguntaSchema = require('../schemas/perguntaSchema');
const { openAiChat } = require('../utils/openAi');
const { User } = require('../models/User');

const router = express.Router();

// Rota de validação de pergunta
router.post(
  '/',
  [
    body('email')
      .isEmail()
      .withMessage(i18n.t('error_required'))
      .custom(async (value) => {
        const userExists = await User.findOne({ email: value });
        if (userExists) {
          throw new Error(i18n.t('error_email_exists'));
        }
        return true;
      }),
  ],
  handleValidationErrors,
  validateWithYup(perguntaSchema),
  (req, res) => {
    res.status(200).json({ mensagem: 'Validação concluída com sucesso.', dados: req.body });
  }
);

// Rota de interação com a IA
router.post('/interagir-com-ia', async (req, res, next) => {
  try {
    const resposta = await openAiChat(req.body.mensagem);
    res.status(200).json({ resposta });
  } catch (err) {
    next(new AiError('Erro ao interagir com a IA', 500));
  }
});

module.exports = router;

5. Função OpenAI (openAi.js)

// utils/openAi.js
const axios = require('axios');

// Função para interagir com a OpenAI (assistente "Gracielle")
const openAiChat = async (mensagem) => {
  const prompt = `Atue como um assistente de atendimento ao cliente da loja G&G Perfumes e Cosméticos, chamada Gracielle. Responda a seguinte pergunta: ${mensagem}`;
  try {
    const resposta = await axios.post('https://api.openai.com/v1/completions', {
      model: 'gpt-3.5-turbo',  // Atualizado para o modelo mais recente
      prompt: prompt,
      max_tokens: 150,
    }, {
      headers: {
        'Authorization': `Bearer ${process.env.OPENAI_API_KEY}`,
      },
    });
    return resposta.data.choices[0].text.trim();
  } catch (error) {
    throw new AiError('Erro ao interagir com a IA', 500);
  }
};

module.exports = { openAiChat };

6. Middleware de Validação de Erros (handleValidationErrors.js)

// middlewares/handleValidationErrors.js
const { validationResult } = require('express-validator');

// Middleware para tratar erros de validação
const handleValidationErrors = (req, res, next) => {
  const errors = validationResult(req);
  if (!errors.isEmpty()) {
    return res.status(400).json({ erros: errors.array().map(err => err.msg) });
  }
  next();
};

module.exports = handleValidationErrors;

7. Validação com Yup (validateWithYup.js)

// middlewares/validateWithYup.js
const validateWithYup = (schema) => {
  return async (req, res, next) => {
    try {
      await schema.validate(req.body, { abortEarly: false });
      next();
    } catch (err) {
      return res.status(400).json({ erros: err.errors });
    }
  };
};

module.exports = validateWithYup;

8. Modelo de Usuário (User.js)

// models/User.js
const mongoose = require('mongoose');

const userSchema = new mongoose.Schema({
  email: { type: String, required: true, unique: true },
  nome: { type: String, required: true },
  senha: { type: String, required: true },
});

const User = mongoose.model('User', userSchema);
module.exports = { User };

9. Arquivos de Tradução (locales/pt/translation.json)

// locales/pt/translation.json
{
  "error_required": "Este campo é obrigatório.",
  "error_email_exists": "Este e-mail já está registrado.",
  "error_past_date": "A data não pode ser no passado.",
  "error_invalid_status": "Status inválido.",
  "error_invalid_cpf": "CPF inválido"
}

10. HTML (index.html)

<!-- public/index.html -->
<!DOCTYPE html>
<html lang="pt">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>G&G Perfumes e Cosméticos</title>
  <link rel="stylesheet" href="assets/styles.css">
  <meta name="csrf-token" content="{{csrfToken}}">
</head>
<body>
  <header>
    <img src="assets/images/logo.png" alt="G&G Perfumes e Cosméticos">
  </header>
</body>
</html>



