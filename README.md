<!DOCTYPE html>
<html lang="pt-BR">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Controle Financeiro</title>
  <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;600&display=swap" rel="stylesheet">
  <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/xlsx/0.18.5/xlsx.full.min.js"></script>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/jspdf/2.5.1/jspdf.umd.min.js"></script>
  <style>
    * { box-sizing: border-box; margin: 0; padding: 0; }
    body {
      font-family: 'Inter', sans-serif;
      background-color: #1e1e2f;
      padding: 40px;
      color: #f0f0f0;
    }
    h1 { text-align: center; margin-bottom: 30px; }
    form, .filtros {
      display: flex;
      flex-wrap: wrap;
      gap: 10px;
      background-color: #2c2c3e;
      padding: 20px;
      border-radius: 12px;
      box-shadow: 0 4px 12px rgba(0, 0, 0, 0.3);
      margin-bottom: 30px;
      max-width: 600px;
      margin-inline: auto;
    }
    input, select, button {
      padding: 10px;
      border: 1px solid #444;
      border-radius: 8px;
      background-color: #1e1e2f;
      color: #f0f0f0;
      flex: 1;
      min-width: 120px;
    }
    button {
      background-color: #4CAF50;
      color: white;
      border: none;
      cursor: pointer;
      transition: background 0.3s;
    }
    button:hover {
      background-color: #45a049;
    }
    #saldo {
      font-size: 24px;
      font-weight: bold;
      text-align: center;
      margin-bottom: 20px;
    }
    .positivo { color: #00e676; }
    .negativo { color: #ff1744; }
    #listaTransacoes {
      max-width: 600px;
      margin: auto;
      background: #2c2c3e;
      padding: 20px;
      border-radius: 12px;
      box-shadow: 0 4px 12px rgba(0, 0, 0, 0.3);
      margin-bottom: 30px;
    }
    .transacao {
      border-bottom: 1px solid #444;
      padding: 10px 0;
    }
    .exportar {
      display: flex;
      justify-content: center;
      gap: 10px;
      margin: 20px auto;
      max-width: 600px;
    }
    canvas {
      background-color: #fff;
      border-radius: 8px;
    }
  </style>
  <script src="https://www.gstatic.com/firebasejs/9.22.1/firebase-app-compat.js"></script>
  <script src="https://www.gstatic.com/firebasejs/9.22.1/firebase-database-compat.js"></script>
</head>
<body>
  <h1>Controle Financeiro</h1>

  <form id="formTransacao">
    <select id="tipo">
      <option value="entrada">Entrada</option>
      <option value="saida">Saída</option>
    </select>
    <input type="number" id="valor" placeholder="Valor (R$)" required>
    <input type="text" id="nome" placeholder="Nome da pessoa" required>
    <input type="date" id="data" required>
    <button type="submit">Adicionar</button>
  </form>

  <div class="filtros">
    <input type="text" id="filtroNome" placeholder="Filtrar por nome">
    <input type="date" id="filtroData">
    <button onclick="exportarExcel()">Exportar Excel</button>
    <button onclick="exportarPDF()">Exportar PDF</button>
  </div>

  <div id="saldo" class="positivo">Saldo: R$ 0,00</div>

  <div id="listaTransacoes"></div>

  <div id="graficoLinhaContainer">
    <h2 style="text-align:center; margin-bottom:15px;">Evolução do Saldo</h2>
    <canvas id="graficoLinha"></canvas>
  </div>

  <div id="graficoPizzaContainer">
    <h2 style="text-align:center; margin-bottom:15px;">Distribuição de Despesas</h2>
    <canvas id="graficoPizza"></canvas>
  </div>

  <script>
    const firebaseConfig = {
      apiKey: "SUA_API_KEY",
      authDomain: "SEU_DOMINIO.firebaseapp.com",
      databaseURL: "https://SEU_PROJETO.firebaseio.com",
      projectId: "SEU_PROJETO",
      storageBucket: "SEU_PROJETO.appspot.com",
      messagingSenderId: "SEU_SENDER_ID",
      appId: "SEU_APP_ID"
    };

    firebase.initializeApp(firebaseConfig);
    const database = firebase.database();

    const form = document.getElementById("formTransacao");
    const lista = document.getElementById("listaTransacoes");
    const saldoEl = document.getElementById("saldo");
    const graficoLinhaCtx = document.getElementById("graficoLinha").getContext("2d");
    let graficoLinha;

    let todasTransacoes = [];

    form.addEventListener("submit", (e) => {
      e.preventDefault();
      const tipo = document.getElementById("tipo").value;
      const valor = parseFloat(document.getElementById("valor").value);
      const nome = document.getElementById("nome").value;
      const data = document.getElementById("data").value;
      const transacao = { tipo, valor, nome, data };
      if (!isNaN(valor) && nome && data) {
        database.ref("transacoes").push(transacao);
      }
      form.reset();
    });

    function atualizarSaldo(transacoes) {
      const filtroNome = document.getElementById("filtroNome").value.toLowerCase();
      const filtroData = document.getElementById("filtroData").value;

      let saldo = 0;
      const saldosAoLongoDoTempo = [];
      const datas = [];

      lista.innerHTML = "";

      const filtradas = transacoes
        .filter(t => (!filtroNome || t.nome.toLowerCase().includes(filtroNome)) &&
                     (!filtroData || t.data === filtroData))
        .sort((a, b) => new Date(a.data) - new Date(b.data));

      filtradas.forEach(t => {
        const valor = parseFloat(t.valor);
        if (!isNaN(valor)) {
          saldo += t.tipo === "entrada" ? valor : -valor;

          datas.push(t.data);
          saldosAoLongoDoTempo.push(saldo);

          const div = document.createElement("div");
          div.className = "transacao";
          div.textContent = `${t.tipo === "entrada" ? 'Entrada' : 'Saída'} - R$ ${valor.toFixed(2)} - ${t.nome} - ${t.data}`;
          lista.appendChild(div);
        }
      });

      saldoEl.textContent = `Saldo: R$ ${saldo.toFixed(2)}`;
      saldoEl.className = saldo >= 0 ? 'positivo' : 'negativo';

      if (graficoLinha) graficoLinha.destroy();
      graficoLinha = new Chart(graficoLinhaCtx, {
        type: 'line',
        data: {
          labels: datas,
          datasets: [{
            label: 'Saldo ao longo do tempo',
            data: saldosAoLongoDoTempo,
            borderColor: '#00e676',
            backgroundColor: 'rgba(0, 230, 118, 0.1)',
            fill: true,
            tension: 0.3
          }]
        },
        options: { scales: { y: { beginAtZero: true } } }
      });
    }

    function exportarExcel() {
      const rows = [['Tipo', 'Valor', 'Nome', 'Data']];
      document.querySelectorAll('.transacao').forEach(div => {
        const partes = div.textContent.split(' - ');
        rows.push(partes);
      });
      const wb = XLSX.utils.book_new();
      const ws = XLSX.utils.aoa_to_sheet(rows);
      XLSX.utils.book_append_sheet(wb, ws, 'Transacoes');
      XLSX.writeFile(wb, 'transacoes.xlsx');
    }

    function exportarPDF() {
      const { jsPDF } = window.jspdf;
      const doc = new jsPDF();
      doc.setFontSize(12);
      let y = 10;
      document.querySelectorAll('.transacao').forEach(div => {
        doc.text(div.textContent, 10, y);
        y += 10;
      });
      doc.save('transacoes.pdf');
    }

    document.getElementById("filtroNome").addEventListener("input", () => atualizarSaldo(todasTransacoes));
    document.getElementById("filtroData").addEventListener("input", () => atualizarSaldo(todasTransacoes));

    database.ref("transacoes").on("value", snapshot => {
      const data = snapshot.val();
      todasTransacoes = data ? Object.values(data) : [];
      atualizarSaldo(todasTransacoes);
    });
  </script>
</body>
</html>
