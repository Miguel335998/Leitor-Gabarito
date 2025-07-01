<!-- CÓDIGO COMPLETO COM DETECÇÃO DA LETRA MARCADA -->
<!DOCTYPE html>
<html lang="pt-BR">
<head>
  <meta charset="UTF-8">
  <title>Correção + Mapeamento</title>
  <script async src="https://docs.opencv.org/4.x/opencv.js" onload="onOpenCvReady();" type="text/javascript"></script>
  <style>
    body {
      font-family: Arial, sans-serif;
      margin: 20px;
    }
    canvas {
      border: 1px solid #ccc;
      margin-top: 10px;
      max-width: 100%;
    }
    #canvasCorrigido {
      width: 800px;
      height: auto;
    }
  </style>
</head>
<body>
  <h2>Correção + Mapeamento</h2>
  <input type="file" id="uploadImagem" accept="image/*">
  <br>
  <canvas id="canvasCorrigido"></canvas>

  <script>
    const canvasCorrigido = document.getElementById('canvasCorrigido');

    function onOpenCvReady() {
      console.log("✅ OpenCV.js carregado");
    }

    function mostrarErro() {
      alert("Erro ao ler a foto. Tire uma que enquadre toda a folha, de cima e bem iluminada.");
    }

    function ordenarPontos(pontos) {
      pontos.sort((a, b) => a.x + a.y - (b.x + b.y));
      let [p0, p1, p2, p3] = pontos;
      let topLeft = p0;
      let bottomRight = p3;
      let rest = [p1, p2];
      let topRight = rest[0].x > rest[1].x ? rest[0] : rest[1];
      let bottomLeft = rest[0].x < rest[1].x ? rest[0] : rest[1];
      return [topLeft, topRight, bottomRight, bottomLeft];
    }

    function mapearQuestoes(img, colunas = 4, linhas = 25, larguraFinal = 800) {
      const alturaFinal = Math.round((larguraFinal * img.rows) / img.cols);
      cv.resize(img, img, new cv.Size(larguraFinal, alturaFinal));

      const larguraColuna = larguraFinal / colunas;
      const alturaLinha = alturaFinal / linhas;

      const mapped = img.clone();
      const cinza = new cv.Mat();
      cv.cvtColor(img, cinza, cv.COLOR_RGBA2GRAY);

      for (let col = 0; col < colunas; col++) {
        for (let lin = 0; lin < linhas; lin++) {
          const x = col * larguraColuna;
          const y = lin * alturaLinha;
          const numeroQuestao = col * linhas + lin + 1;

          const p1 = new cv.Point(x, y);
          const p2 = new cv.Point(x + larguraColuna, y + alturaLinha);
          cv.rectangle(mapped, p1, p2, new cv.Scalar(0, 255, 0, 255), 2);

          const partes = 6;
          const larguraParte = larguraColuna / partes;

          // Linhas verticais amarelas
          for (let i = 1; i < partes; i++) {
            const xLinha = x + i * larguraParte;
            const pt1 = new cv.Point(xLinha, y);
            const pt2 = new cv.Point(xLinha, y + alturaLinha);
            cv.line(mapped, pt1, pt2, new cv.Scalar(255, 255, 0, 255), 2);
          }

          // Análise do preenchimento das alternativas (A a E)
          let menores = 255; // maior valor possível no cinza
          let alternativa = "-";
          let letras = ["A", "B", "C", "D", "E"];

          for (let i = 1; i < partes; i++) {
            let roiX = Math.floor(x + i * larguraParte);
            let roiY = Math.floor(y);
            let roiW = Math.floor(larguraParte);
            let roiH = Math.floor(alturaLinha);
            let roi = cinza.roi(new cv.Rect(roiX, roiY, roiW, roiH));

            let media = cv.mean(roi)[0];
            roi.delete();

            if (media < menores) {
              menores = media;
              alternativa = letras[i - 1]; // i começa em 1
            }
          }

          console.log(`Questão ${numeroQuestao} (coluna ${col + 1}, linha ${lin + 1}) - Letra ${alternativa}`);
        }
      }

      canvasCorrigido.width = larguraFinal;
      canvasCorrigido.height = alturaFinal;
      cv.imshow('canvasCorrigido', mapped);
      mapped.delete();
      cinza.delete();
    }

    document.getElementById('uploadImagem').addEventListener('change', function (e) {
      const file = e.target.files[0];
      if (!file) return;

      const reader = new FileReader();
      reader.onload = function (event) {
        const img = new Image();
        img.onload = function () {
          const canvasTemp = document.createElement("canvas");
          canvasTemp.width = img.width;
          canvasTemp.height = img.height;
          const ctx = canvasTemp.getContext('2d');
          ctx.drawImage(img, 0, 0);

          let src = cv.imread(canvasTemp);
          let gray = new cv.Mat();
          let blurred = new cv.Mat();
          let edged = new cv.Mat();
          let contours = new cv.MatVector();
          let hierarchy = new cv.Mat();

          cv.cvtColor(src, gray, cv.COLOR_RGBA2GRAY);
          cv.GaussianBlur(gray, blurred, new cv.Size(5, 5), 0);
          cv.Canny(blurred, edged, 50, 150);
          cv.findContours(edged, contours, hierarchy, cv.RETR_EXTERNAL, cv.CHAIN_APPROX_SIMPLE);

          let biggest = null;
          let biggestArea = 0;

          for (let i = 0; i < contours.size(); i++) {
            let cnt = contours.get(i);
            let peri = cv.arcLength(cnt, true);
            let approx = new cv.Mat();
            cv.approxPolyDP(cnt, approx, 0.02 * peri, true);

            if (approx.rows === 4) {
              let area = cv.contourArea(cnt);
              if (area > biggestArea) {
                biggestArea = area;
                if (biggest) biggest.delete();
                biggest = approx;
              } else {
                approx.delete();
              }
            } else {
              approx.delete();
            }
          }

          if (biggest) {
            let pontos = [];
            for (let i = 0; i < 4; i++) {
              let pt = biggest.intPtr(i);
              pontos.push(new cv.Point(pt[0], pt[1]));
            }

            let [tl, tr, br, bl] = ordenarPontos(pontos);

            let larguraTopo = Math.hypot(tr.x - tl.x, tr.y - tl.y);
            let larguraBase = Math.hypot(br.x - bl.x, br.y - bl.y);
            let alturaEsq = Math.hypot(bl.x - tl.x, bl.y - tl.y);
            let alturaDir = Math.hypot(br.x - tr.x, br.y - tr.y);

            let difH = Math.abs(larguraTopo - larguraBase);
            let difV = Math.abs(alturaEsq - alturaDir);

            if (difH > 300 || difV > 300 || larguraTopo < 60 || alturaEsq < 60) {
              mostrarErro();
              src.delete(); gray.delete(); blurred.delete(); edged.delete();
              contours.delete(); hierarchy.delete(); biggest.delete();
              return;
            }

            let maxWidth = Math.max(larguraBase, larguraTopo);
            let maxHeight = Math.max(
              Math.hypot(tr.x - br.x, tr.y - br.y),
              Math.hypot(tl.x - bl.x, tl.y - bl.y)
            );

            let srcQuad = cv.matFromArray(4, 1, cv.CV_32FC2, [
              tl.x, tl.y,
              tr.x, tr.y,
              br.x, br.y,
              bl.x, bl.y
            ]);
            let dstQuad = cv.matFromArray(4, 1, cv.CV_32FC2, [
              0, 0,
              maxWidth, 0,
              maxWidth, maxHeight,
              0, maxHeight
            ]);

            let M = cv.getPerspectiveTransform(srcQuad, dstQuad);
            let warped = new cv.Mat();
            let dsize = new cv.Size(maxWidth, maxHeight);
            cv.warpPerspective(src, warped, M, dsize, cv.INTER_LINEAR, cv.BORDER_CONSTANT, new cv.Scalar());

            mapearQuestoes(warped);

            warped.delete(); M.delete(); srcQuad.delete(); dstQuad.delete();
          } else {
            mostrarErro();
          }

          src.delete(); gray.delete(); blurred.delete(); edged.delete();
          contours.delete(); hierarchy.delete();
          if (biggest) biggest.delete();
        };
        img.src = event.target.result;
      };
      reader.readAsDataURL(file);
    });
  </script>
</body>
</html>
