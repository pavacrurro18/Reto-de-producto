app.post('/webhook/whatsapp', async (req, res) => {
  try {
    const messageText: string | undefined = req.body?.message || req.body?.text || req.body?.Body;
    if (!messageText) {
      return res.status(400).json({ error: 'Missing message text' });
    }

    const result = await classifyLead(messageText);
    return res.json(result);
  } catch (error) {
    console.error('Webhook error:', error);
    return res.status(500).json({ error: 'Internal server error' });
  }
});

export async function classifyLead(message: string): Promise<LeadScoringResult> {
  const openai = getOpenAI();

  const system =
    'Eres un asistente de ventas. Clasifica leads entrantes por su intención de compra en: "caliente" (alta intención, listo para comprar), "tibio" (interés moderado, necesita más info), "frio" (bajo interés o fuera de perfil). Devuelve JSON estricto con keys: score (caliente|tibio|frio), confidence (0-1), reasons (array corta).';

  const user = `Mensaje de WhatsApp del lead:
"""
${message}
"""
Tarea: clasifica el lead.`;

  const completion = await openai.chat.completions.create({
    model: process.env.OPENAI_MODEL || 'gpt-4o-mini',
    messages: [
      { role: 'system', content: system },
      { role: 'user', content: user }
    ],
    response_format: { type: 'json_object' }
  });

  const content = completion.choices[0]?.message?.content ?? '{}';
  try {
    const parsed = JSON.parse(content);
    const score = String(parsed.score || '').toLowerCase();
    const mapped: LeadScore = score === 'caliente' ? 'caliente' : score === 'tibio' ? 'tibio' : 'frio';
    const confidence = typeof parsed.confidence === 'number' ? Math.max(0, Math.min(1, parsed.confidence)) : 0.6;
    const reasons = Array.isArray(parsed.reasons) ? reasons = parsed.reasons.slice(0, 4) : [];
    return { score: mapped, confidence, reasons };
  } catch (_e) {
    return { score: 'frio', confidence: 0.5, reasons: ['Respuesta no parseable'] };
  }
}
