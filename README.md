(function() {
    // 1. Remove versão anterior se já estiver aberta na página
    const antigo = document.getElementById('painel-senhas-sabin-v2');
    if (antigo) antigo.remove();

    // 2. Configuração de Rótulos e Cores por Prioridade
    const PRIORIDADE = {
        'A': { label: '80+ALTA', cor: '#e74c3c', corFundo: '#2c0a0a', icone: '🔴' },
        'P': { label: 'PREFER.', cor: '#e67e22', corFundo: '#2c1a0a', icone: '🟠' },
        'G': { label: 'GERAL', cor: '#95a5a6', corFundo: '#1a1a1a', icone: '⚪' }
    };

    // 3. Tempo limite (SLA em minutos) por tipo de atendimento
    const TIPOS = {
        'P': { label: 'Pendência', tempo: 12 },
        'V': { label: 'Vacina', tempo: 12 },
        'R': { label: 'Resultado', tempo: 12 },
        'E': { label: 'Exames', tempo: 20 },
        'A': { label: 'Agendamento', tempo: 15 },
        'D': { label: 'Digital', tempo: 12 }
    };

    let baseDados = [];

    // 4. Criação do elemento container do Painel
    const painel = document.createElement('div');
    painel.id = 'painel-senhas-sabin-v2';
    painel.style.cssText = "position:fixed;top:10px;right:10px;width:320px;background:#18181b;color:#fff;border-radius:12px;box-shadow:0 8px 32px rgba(0,0,0,.85);z-index:2147483647;font-family:system-ui,-apple-system,sans-serif;border:1px solid #3f3f46;overflow:hidden;";

    painel.innerHTML = `
        <div style="background:#27272a;padding:12px 14px;display:flex;align-items:center;justify-content:space-between;border-bottom:1px solid #3f3f46;">
            <span style="font-weight:700;font-size:14px;display:flex;align-items:center;gap:6px;">📋 Painel de Senhas</span>
            <div style="display:flex;gap:6px;">
                <button id="psb-atualizar" style="background:#3b82f6;border:none;color:#fff;border-radius:6px;width:28px;height:24px;cursor:pointer;font-size:14px;display:flex;align-items:center;justify-content:center;" title="Atualizar">↻</button>
                <button id="psb-fechar" style="background:#52525b;border:none;color:#fff;border-radius:6px;width:28px;height:24px;cursor:pointer;font-size:12px;display:flex;align-items:center;justify-content:center;">✕</button>
            </div>
        </div>
        <div style="padding:12px 14px;background:#18181b;border-bottom:1px solid #27272a;display:flex;gap:8px;">
            <div style="flex:1;text-align:center;background:#000;border:1px solid #e74c3c;border-radius:8px;padding:6px 0;">
                <div style="font-size:18px;font-weight:700;color:#e74c3c;" id="psb-num-A">0</div>
                <div style="font-size:9px;color:#a1a1aa;margin-top:2px;">🔴 80+ALTA</div>
            </div>
            <div style="flex:1;text-align:center;background:#000;border:1px solid #e67e22;border-radius:8px;padding:6px 0;">
                <div style="font-size:18px;font-weight:700;color:#e67e22;" id="psb-num-P">0</div>
                <div style="font-size:9px;color:#a1a1aa;margin-top:2px;">🟠 PREFER.</div>
            </div>
            <div style="flex:1;text-align:center;background:#000;border:1px solid #52525b;border-radius:8px;padding:6px 0;">
                <div style="font-size:18px;font-weight:700;color:#d4d4d8;" id="psb-num-G">0</div>
                <div style="font-size:9px;color:#a1a1aa;margin-top:2px;">⚪ GERAL</div>
            </div>
        </div>
        <div id="psb-recomendada" style="display:none;padding:12px 14px;background:#052e16;border-bottom:1px solid #27272a;">
            <div style="font-size:11px;color:#4ade80;font-weight:700;margin-bottom:8px;display:flex;align-items:center;gap:4px;">☑ PRÓXIMA RECOMENDADA</div>
            <div id="psb-recom-corpo"></div>
        </div>
        <div id="psb-lista" style="max-height:360px;overflow-y:auto;padding:10px 14px;background:#18181b;display:flex;flex-direction:column;gap:6px;"></div>
        <div style="padding:8px 14px;text-align:center;font-size:10px;color:#71717a;background:#18181b;">Leitura: <span id="psb-hora">--</span></div>
    `;

    document.body.appendChild(painel);

    function fmt(seg) {
        seg = Math.max(0, Math.floor(seg));
        const h = Math.floor(seg / 3600);
        const m = Math.floor((seg % 3600) / 60);
        const s = seg % 60;
        return [h, m, s].map(n => String(n).padStart(2, '0')).join(':');
    }

    function lerSenhas() {
        const emAtendimento = new Set();
        Array.from(document.querySelectorAll('div')).forEach(el => {
            if (el.closest('#painel-senhas-sabin-v2')) return;
            const txt = el.innerText || '';
            if (txt.length > 500) return;
            if (!/TEMPO DE ATENDIMENTO/i.test(txt)) return;
            (txt.match(/[A-Z]{1,2}\d{3}/g) || []).forEach(s => emAtendimento.add(s));
        });

        const IGNORAR = /TEMPO DE ATENDIMENTO|Finalizado pelo|Concluída|Histórico/i;
        const vistos = new Set();
        const novas = [];

        Array.from(document.querySelectorAll('div,li')).forEach(el => {
            if (el.closest('#painel-senhas-sabin-v2')) return;
            const txt = (el.innerText || '').trim();
            if (txt.length < 10 || txt.length > 150) return;
            if (!/[A-Z]{1,2}\d{3}/.test(txt)) return;
            if (!/\d{2}:\d{2}:\d{2}/.test(txt)) return;
            if (IGNORAR.test(txt)) return;

            const senhasNoCard = new Set((txt.match(/[A-Z]{1,2}\d{3}/g) || []));
            if (senhasNoCard.size !== 1) return;

            const codSenha = [...senhasNoCard][0];
            if (vistos.has(codSenha) || emAtendimento.has(codSenha)) return;

            const tempoMatch = txt.match(/(\d{2}:\d{2}:\d{2})/);
            if (!tempoMatch) return;

            vistos.add(codSenha);

            const partes = tempoMatch[1].split(':').map(Number);
            const segundos = partes[0] * 3600 + partes[1] * 60 + partes[2];

            const prefixo = codSenha.replace(/\d/g, '');
            const priorLetra = prefixo.slice(-1);
            if (!['A', 'P', 'G'].includes(priorLetra)) return;

            const tipoLetra = prefixo.length > 1 ? prefixo[0] : prefixo;
            const tipoObj = TIPOS[tipoLetra] || { label: tipoLetra, tempo: 12 };

            const anterior = baseDados.find(b => b.senha === codSenha);

            novas.push({
                senha: codSenha,
                prioridade: priorLetra,
                tipoLabel: tipoObj.label,
                tempoLimiteMin: tipoObj.tempo,
                segundosBase: anterior ? anterior.segundosBase : segundos,
                timestampBase: anterior ? anterior.timestampBase : Date.now() - segundos * 1000
            });
        });

        novas.sort((a, b) => {
            const pctA = (((Date.now() - a.timestampBase) / 1000) / (a.tempoLimiteMin * 60)) * 100;
            const pctB = (((Date.now() - b.timestampBase) / 1000) / (b.tempoLimiteMin * 60)) * 100;
            return pctB - pctA;
        });

        baseDados = novas;

        ['A', 'P', 'G'].forEach(p => {
            const countEl = document.getElementById(`psb-num-${p}`);
            if (countEl) countEl.innerText = novas.filter(s => s.prioridade === p).length;
        });

        renderizarLista();
        const horaEl = document.getElementById('psb-hora');
        if (horaEl) horaEl.innerText = new Date().toLocaleTimeString('pt-BR');
    }

    function renderizarLista() {
        const listaEl = document.getElementById('psb-lista');
        const recomendadaEl = document.getElementById('psb-recomendada');

        if (baseDados.length > 0) {
            recomendadaEl.style.display = 'block';
            const top = baseDados[0];
            const cfgTop = PRIORIDADE[top.prioridade];
            const segTop = (Date.now() - top.timestampBase) / 1000;
            const pctTop = Math.min((segTop / (top.tempoLimiteMin * 60)) * 100, 999);

            const recomCorpo = document.getElementById('psb-recom-corpo');
            if (recomCorpo) {
                recomCorpo.innerHTML = `
                    <div style="display:flex;align-items:center;gap:12px;">
                        <div style="font-size:24px;">${cfgTop.icone}</div>
                        <div style="flex:1;">
                            <div style="font-weight:700;font-size:18px;color:#fff;line-height:1;">${top.senha}</div>
                            <div style="font-size:10px;color:#a1a1aa;margin-top:4px;">${top.tipoLabel} - ${cfgTop.label} - <span id="psb-recom-tempo">⏱ ${fmt(segTop)}</span></div>
                        </div>
                        <div style="text-align:right;">
                            <div id="psb-recom-pct" style="font-size:16px;font-weight:700;color:${pctTop >= 100 ? '#ef4444' : '#22c55e'};">${Math.round(pctTop)}%</div>
                            <div style="font-size:9px;color:#71717a;">do limite</div>
                        </div>
                    </div>
                `;
            }
        } else {
            recomendadaEl.style.display = 'none';
        }

        if (baseDados.length === 0) {
            listaEl.innerHTML = '<div style="text-align:center;color:#71717a;padding:20px;font-size:12px;">Nenhuma senha na fila.</div>';
            return;
        }

        listaEl.innerHTML = baseDados.map((s, idx) => {
            const cfg = PRIORIDADE[s.prioridade];
            const segAtual = (Date.now() - s.timestampBase) / 1000;
            const pct = Math.min((segAtual / (s.tempoLimiteMin * 60)) * 100, 999);
            const urgente = pct >= 100;

            return `
                <div data-senha="${s.senha}" style="display:flex;align-items:center;gap:10px;background:#27272a;border:1px solid ${urgente ? '#ea580c' : '#3f3f46'};border-radius:8px;padding:10px 12px;${urgente ? 'box-shadow:0 0 10px rgba(234,88,12,0.1);' : ''}">
                    <div style="font-size:10px;color:#71717a;width:12px;text-align:right;">${idx + 1}º</div>
                    <div style="font-size:16px;">${cfg.icone}</div>
                    <div style="flex:1;">
                        <div style="font-weight:700;font-size:14px;color:${cfg.cor};line-height:1;">${s.senha}</div>
                        <div style="font-size:10px;color:#a1a1aa;margin-top:4px;">${s.tipoLabel} - ${cfg.label}</div>
                    </div>
                    <div style="text-align:right;">
                        <div class="psb-tempo-val" style="font-size:12px;color:${urgente ? '#ea580c' : '#a1a1aa'};font-weight:700;">⏱ ${fmt(segAtual)}</div>
                        <div class="psb-pct-val" style="font-size:11px;color:${pct >= 100 ? '#ef4444' : pct >= 80 ? '#f97316' : '#71717a'};font-weight:700;margin-top:2px;">${Math.round(pct)}%</div>
                    </div>
                </div>
            `;
        }).join('');
    }

    function tick() {
        if (!document.getElementById('painel-senhas-sabin-v2')) return;

        document.querySelectorAll('#psb-lista [data-senha]').forEach(card => {
            const codSenha = card.getAttribute('data-senha');
            const s = baseDados.find(item => item.senha === codSenha);
            if (!s) return;

            const segAtual = (Date.now() - s.timestampBase) / 1000;
            const pct = Math.min((segAtual / (s.tempoLimiteMin * 60)) * 100, 999);
            const urgente = pct >= 100;

            const elTempo = card.querySelector('.psb-tempo-val');
            const elPct = card.querySelector('.psb-pct-val');

            if (elTempo) {
                elTempo.innerText = '⏱ ' + fmt(segAtual);
                elTempo.style.color = urgente ? '#ea580c' : '#a1a1aa';
            }
            if (elPct) {
                elPct.innerText = Math.round(pct) + '%';
                elPct.style.color = pct >= 100 ? '#ef4444' : pct >= 80 ? '#f97316' : '#71717a';
            }

            card.style.borderColor = urgente ? '#ea580c' : '#3f3f46';
            card.style.boxShadow = urgente ? '0 0 10px rgba(234,88,12,0.1)' : 'none';
        });

        if (baseDados.length > 0) {
            const top = baseDados[0];
            const segTop = (Date.now() - top.timestampBase) / 1000;
            const pctTop = Math.min((segTop / (top.tempoLimiteMin * 60)) * 100, 999);

            const elRT = document.getElementById('psb-recom-tempo');
            const elRP = document.getElementById('psb-recom-pct');

            if (elRT) elRT.innerText = '⏱ ' + fmt(segTop);
            if (elRP) {
                elRP.innerText = Math.round(pctTop) + '%';
                elRP.style.color = pctTop >= 100 ? '#ef4444' : '#22c55e';
            }
        }
    }

    document.getElementById('psb-atualizar').onclick = lerSenhas;
    document.getElementById('psb-fechar').onclick = () => {
        document.getElementById('painel-senhas-sabin-v2').remove();
    };

    lerSenhas();

    const intLer = setInterval(() => {
        if (!document.getElementById('painel-senhas-sabin-v2')) {
            clearInterval(intLer);
            return;
        }
        lerSenhas();
    }, 5000);

    const intTick = setInterval(() => {
        if (!document.getElementById('painel-senhas-sabin-v2')) {
            clearInterval(intTick);
            return;
        }
        tick();
    }, 1000);
})();
