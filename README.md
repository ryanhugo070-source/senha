(() => {
    // Remove versão anterior se já estiver aberta
    if (document.getElementById('painel-senhas-sabin-v2')) {
        document.getElementById('painel-senhas-sabin-v2').remove();
    }

    // Configuração de cores e rótulos por Prioridade (visual)
    const PRIORIDADE = {
        'A': { label: '80+ ALTA',     cor: '#e74c3c', corFundo: '#2c0a0a', icone: '🔴' },
        'P': { label: 'PREFERENCIAL', cor: '#e67e22', corFundo: '#2c1a0a', icone: '🟠' },
        'G': { label: 'GERAL',        cor: '#95a5a6', corFundo: '#1a1a1a', icone: '⚪' }
    };

    // ⏱️ TEMPO LIMITE (SLA) DEFINIDO POR TIPO DE SERVIÇO (em minutos)
    // Altere os valores abaixo conforme a necessidade da sua unidade:
    const TIPOS = { 
        'P': { label: 'Pendência',   tempo: 12 }, // 10 min
        'V': { label: 'Vacina',      tempo: 12 }, // 12 min
        'R': { label: 'Resultado',   tempo: 12 },  // 12 min
        'E': { label: 'Exames',      tempo: 15 }, // 15 min (Realiza Exames)
        'A': { label: 'Agendamento', tempo: 12 }, // 12 min
        'D': { label: 'Digital',     tempo: 12 }  // 12 min
    };

    // Estado global dos dados e filtros
    let baseDados = [];
    const filtros = {
        busca: '',
        prioridade: 'TODAS',
        tipo: 'TODOS',
        tempo: 'TODOS'
    };

    // ── ESTRUTURA DO PAINEL ─────────────────────────────────────────
    const painel = document.createElement('div');
    painel.id = 'painel-senhas-sabin-v2';
    painel.style.cssText = `
        position:fixed; top:10px; right:10px; width:340px;
        background:#111; color:#fff; border-radius:12px;
        box-shadow:0 8px 32px rgba(0,0,0,.85); z-index:2147483647;
        font-family:'Segoe UI',Arial,sans-serif; border:1px solid #333; overflow:hidden;
    `;

    painel.innerHTML = `
        <!-- Cabeçalho -->
        <div style="background:#1a1a1a;padding:10px 14px;display:flex;align-items:center;justify-content:space-between;border-bottom:1px solid #333;">
            <span style="font-weight:700;font-size:13px;display:flex;align-items:center;gap:6px;">
                📋 Painel por Tipo de Serviço
            </span>
            <div style="display:flex;gap:6px;">
                <button id="psb-atualizar" style="background:#2d7dff;border:none;color:#fff;border-radius:6px;padding:3px 9px;cursor:pointer;font-size:12px;" title="Atualizar dados">↻</button>
                <button id="psb-fechar" style="background:#444;border:none;color:#fff;border-radius:6px;padding:3px 7px;cursor:pointer;font-size:12px;">✕</button>
            </div>
        </div>

        <!-- Contadores Totais -->
        <div style="padding:8px 10px;background:#161616;border-bottom:1px solid #2a2a2a;display:flex;gap:6px;">
            <div style="flex:1;text-align:center;background:#2c0a0a;border:1px solid #e74c3c;border-radius:6px;padding:4px;">
                <div style="font-size:16px;font-weight:700;color:#e74c3c;" id="psb-num-A">0</div>
                <div style="font-size:8px;color:#e74c3c;">🔴 80+ ALTA</div>
            </div>
            <div style="flex:1;text-align:center;background:#2c1a0a;border:1px solid #e67e22;border-radius:6px;padding:4px;">
                <div style="font-size:16px;font-weight:700;color:#e67e22;" id="psb-num-P">0</div>
                <div style="font-size:8px;color:#e67e22;">🟠 PREFER.</div>
            </div>
            <div style="flex:1;text-align:center;background:#1a1a1a;border:1px solid #555;border-radius:6px;padding:4px;">
                <div style="font-size:16px;font-weight:700;color:#aaa;" id="psb-num-G">0</div>
                <div style="font-size:8px;color:#aaa;">⚪ GERAL</div>
            </div>
        </div>

        <!-- BARRA DE FILTROS -->
        <div style="padding:8px 10px;background:#181818;border-bottom:1px solid #2a2a2a;display:flex;flex-direction:column;gap:6px;">
            <!-- Pesquisa por código -->
            <input type="text" id="psb-filtro-busca" placeholder="🔍 Buscar código (ex: EG150)..." 
                style="width:100%;box-sizing:border-box;background:#0d0d0d;border:1px solid #333;color:#fff;border-radius:6px;padding:5px 8px;font-size:11px;outline:none;">
            
            <!-- Seletores lado a lado -->
            <div style="display:flex;gap:4px;">
                <select id="psb-filtro-prioridade" style="flex:1;background:#0d0d0d;color:#ccc;border:1px solid #333;border-radius:4px;padding:4px;font-size:10px;outline:none;">
                    <option value="TODAS">Prioridade: Todas</option>
                    <option value="A">🔴 80+ ALTA</option>
                    <option value="P">🟠 PREFERENCIAL</option>
                    <option value="G">⚪ GERAL</option>
                </select>

                <select id="psb-filtro-tipo" style="flex:1;background:#0d0d0d;color:#ccc;border:1px solid #333;border-radius:4px;padding:4px;font-size:10px;outline:none;">
                    <option value="TODOS">Serviço: Todos</option>
                    ${Object.values(TIPOS).map(t => `<option value="${t.label}">${t.label}</option>`).join('')}
                </select>
            </div>

            <select id="psb-filtro-tempo" style="width:100%;background:#0d0d0d;color:#ccc;border:1px solid #333;border-radius:4px;padding:4px;font-size:10px;outline:none;">
                <option value="TODOS">⏱ Tempo: Todos os status</option>
                <option value="URGENTE">🔥 Crítico / Alerta (&gt;80% do limite)</option>
                <option value="ESTOURADO">🚨 Limite Excedido (&gt;100%)</option>
                <option value="MAIOR_10">&gt; 10 minutos de espera</option>
                <option value="MAIOR_20">&gt; 20 minutos de espera</option>
            </select>
        </div>

        <!-- Card da Recomendada -->
        <div id="psb-recomendada" style="display:none;padding:8px 10px;background:#0a1f0a;border-bottom:1px solid #2a2a2a;">
            <div style="font-size:9px;color:#2ecc71;font-weight:700;margin-bottom:3px;">✅ PRÓXIMA RECOMENDADA</div>
            <div id="psb-recom-corpo"></div>
        </div>

        <!-- Lista de Senhas -->
        <div id="psb-lista" style="max-height:360px;overflow-y:auto;padding:6px;"></div>

        <!-- Rodapé -->
        <div style="padding:5px 10px;display:flex;justify-content:space-between;align-items:center;font-size:9px;color:#555;border-top:1px solid #1a1a1a;background:#0d0d0d;">
            <span>Exibindo: <strong id="psb-qtd-visivel" style="color:#aaa;">0</strong> senhas</span>
            <span>Leitura: <span id="psb-hora">--</span></span>
        </div>
    `;

    document.body.appendChild(painel);

    const style = document.createElement('style');
    style.id = 'psb-styles';
    style.innerHTML = `@keyframes psb-pisca{0%,100%{opacity:1}50%{opacity:.35}}`;
    document.head.appendChild(style);

    // ── FORMATAÇÃO DE TEMPO ────────────────────────────────────────
    function fmt(seg) {
        seg = Math.max(0, Math.floor(seg));
        const h = Math.floor(seg / 3600);
        const m = Math.floor((seg % 3600) / 60);
        const s = seg % 60;
        return [h, m, s].map(n => String(n).padStart(2, '0')).join(':');
    }

    // ── FILTRAGEM DOS DADOS ────────────────────────────────────────
    function obterSenhasFiltradas() {
        return baseDados.filter(s => {
            const segAtual = (Date.now() - s.timestampBase) / 1000;
            const pct = (segAtual / (s.tempoLimiteMin * 60)) * 100;

            // Filtro Texto
            if (filtros.busca && !s.senha.toLowerCase().includes(filtros.busca.toLowerCase())) return false;

            // Filtro Prioridade
            if (filtros.prioridade !== 'TODAS' && s.prioridade !== filtros.prioridade) return false;

            // Filtro Tipo de Serviço
            if (filtros.tipo !== 'TODOS' && s.tipoLabel !== filtros.tipo) return false;

            // Filtro de Tempo / SLA
            if (filtros.tempo === 'URGENTE' && pct < 80) return false;
            if (filtros.tempo === 'ESTOURADO' && pct < 100) return false;
            if (filtros.tempo === 'MAIOR_10' && segAtual < 600) return false;
            if (filtros.tempo === 'MAIOR_20' && segAtual < 1200) return false;

            return true;
        });
    }

    // ── LEITURA DA TELA ────────────────────────────────────────────
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

            const tipoObj = TIPOS[tipoLetra] || { label: tipoLetra, tempo: 15 };
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

        // Ordena por % do tempo limite do serviço correspondente
        novas.sort((a, b) => {
            const pctA = (((Date.now() - a.timestampBase) / 1000) / (a.tempoLimiteMin * 60)) * 100;
            const pctB = (((Date.now() - b.timestampBase) / 1000) / (b.tempoLimiteMin * 60)) * 100;
            return pctB - pctA;
        });

        baseDados = novas;

        // Contadores por prioridade no topo
        ['A', 'P', 'G'].forEach(p => {
            const countEl = document.getElementById(`psb-num-${p}`);
            if (countEl) countEl.innerText = novas.filter(s => s.prioridade === p).length;
        });

        renderizarLista();
        document.getElementById('psb-hora').innerText = new Date().toLocaleTimeString('pt-BR');
    }

    // ── RENDERIZAÇÃO DA LISTA ──────────────────────────────────────
    function renderizarLista() {
        const listaEl = document.getElementById('psb-lista');
        const recomendadaEl = document.getElementById('psb-recomendada');
        const filtradas = obterSenhasFiltradas();

        document.getElementById('psb-qtd-visivel').innerText = `${filtradas.length}/${baseDados.length}`;

        // Recomendada Principal
        if (baseDados.length > 0) {
            recomendadaEl.style.display = 'block';
            const top = baseDados[0];
            const cfgTop = PRIORIDADE[top.prioridade];
            const segTop = (Date.now() - top.timestampBase) / 1000;
            const pctTop = Math.min((segTop / (top.tempoLimiteMin * 60)) * 100, 999);
            
            document.getElementById('psb-recom-corpo').innerHTML = `
                <div style="display:flex;align-items:center;gap:8px;">
                    <div style="font-size:20px;">${cfgTop.icone}</div>
                    <div style="flex:1;">
                        <div style="font-weight:700;font-size:15px;color:#fff;">${top.senha}</div>
                        <div style="font-size:10px;color:#888;">${top.tipoLabel} · ${cfgTop.label} · <span id="psb-recom-tempo">⏱ ${fmt(segTop)}</span></div>
                    </div>
                    <div style="text-align:right;">
                        <div id="psb-recom-pct" style="font-size:14px;font-weight:700;color:${pctTop >= 100 ? '#e74c3c' : '#2ecc71'};">${Math.round(pctTop)}%</div>
                        <div style="font-size:8px;color:#555;">do limite (${top.tempoLimiteMin}m)</div>
                    </div>
                </div>`;
        } else {
            recomendadaEl.style.display = 'none';
        }

        // Renderiza lista filtrada
        if (filtradas.length === 0) {
            listaEl.innerHTML = '<div style="text-align:center;color:#555;padding:20px;font-size:11px;">Nenhuma senha encontrada com os filtros selecionados.</div>';
            return;
        }

        listaEl.innerHTML = filtradas.map((s, idx) => {
            const cfg = PRIORIDADE[s.prioridade];
            const segAtual = (Date.now() - s.timestampBase) / 1000;
            const pct = Math.min((segAtual / (s.tempoLimiteMin * 60)) * 100, 999);
            const urgente = pct >= 80;

            return `
            <div data-senha="${s.senha}" style="display:flex;align-items:center;gap:8px;
                background:${cfg.corFundo};border:1px solid ${cfg.cor}44;border-left:3px solid ${cfg.cor};
                border-radius:8px;padding:7px 9px;margin-bottom:5px;
                ${urgente ? `animation:psb-pisca 1s infinite;box-shadow:0 0 8px ${cfg.cor};` : ''}">
                <div style="font-size:10px;color:#555;width:14px;">${idx + 1}º</div>
                <div style="font-size:15px;">${cfg.icone}</div>
                <div style="flex:1;">
                    <div style="font-weight:700;font-size:14px;color:${cfg.cor};">${s.senha}</div>
                    <div style="font-size:10px;color:#777;">${s.tipoLabel} · ${cfg.label} (${s.tempoLimiteMin}m)</div>
                </div>
                <div style="text-align:right;">
                    <div class="psb-tempo-val" style="font-size:11px;color:${urgente ? cfg.cor : '#aaa'};font-weight:${urgente ? '700' : '400'};">⏱ ${fmt(segAtual)}</div>
                    <div class="psb-pct-val" style="font-size:10px;color:${pct >= 100 ? '#e74c3c' : pct >= 80 ? '#e67e22' : '#666'};font-weight:700;">${Math.round(pct)}%</div>
                </div>
            </div>`;
        }).join('');
    }

    // ── CRONÔMETRO VIVO (1 segundo) ────────────────────────────────
    function tick() {
        if (!document.getElementById('painel-senhas-sabin-v2')) return;

        document.querySelectorAll('#psb-lista [data-senha]').forEach(card => {
            const codSenha = card.getAttribute('data-senha');
            const s = baseDados.find(item => item.senha === codSenha);
            if (!s) return;

            const cfg = PRIORIDADE[s.prioridade];
            const segAtual = (Date.now() - s.timestampBase) / 1000;
            const pct = Math.min((segAtual / (s.tempoLimiteMin * 60)) * 100, 999);

            const elTempo = card.querySelector('.psb-tempo-val');
            const elPct = card.querySelector('.psb-pct-val');

            if (elTempo) elTempo.innerText = '⏱ ' + fmt(segAtual);
            if (elPct) {
                elPct.innerText = Math.round(pct) + '%';
                elPct.style.color = pct >= 100 ? '#e74c3c' : pct >= 80 ? '#e67e22' : '#666';
            }
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
                elRP.style.color = pctTop >= 100 ? '#e74c3c' : '#2ecc71';
            }
        }
    }

    // ── EVENTOS ──────────────────────────────────────────────────
    document.getElementById('psb-filtro-busca').oninput = (e) => {
        filtros.busca = e.target.value.trim();
        renderizarLista();
    };

    document.getElementById('psb-filtro-prioridade').onchange = (e) => {
        filtros.prioridade = e.target.value;
        renderizarLista();
    };

    document.getElementById('psb-filtro-tipo').onchange = (e) => {
        filtros.tipo = e.target.value;
        renderizarLista();
    };

    document.getElementById('psb-filtro-tempo').onchange = (e) => {
        filtros.tempo = e.target.value;
        renderizarLista();
    };

    document.getElementById('psb-atualizar').onclick = lerSenhas;
    document.getElementById('psb-fechar').onclick = () => {
        document.getElementById('painel-senhas-sabin-v2').remove();
    };

    // ── EXECUÇÃO E INTERVALOS ──────────────────────────────────────
    lerSenhas();

    const intLer = setInterval(() => {
        if (!document.getElementById('painel-senhas-sabin-v2')) { clearInterval(intLer); return; }
        lerSenhas();
    }, 5000);

    const intTick = setInterval(() => {
        if (!document.getElementById('painel-senhas-sabin-v2')) { clearInterval(intTick); return; }
        tick();
    }, 1000);
})();
