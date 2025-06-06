<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8" />
  <title>Confirmación de Asistencia</title>
  <style>
    /* Estilos básicos para que se vea ordenado */
    body {
      font-family: Arial, sans-serif;
      max-width: 500px;
      margin: 20px auto;
      padding: 0 10px;
    }
    label {
      display: block;
      margin-top: 10px;
      font-weight: bold;
    }
    select, input[type="text"], input[type="number"], textarea, button {
      width: 100%;
      padding: 8px;
      margin-top: 5px;
      box-sizing: border-box;
    }
    button {
      margin-top: 15px;
      background-color: #003366;
      color: white;
      border: none;
      cursor: pointer;
      font-size: 16px;
    }
    button:hover {
      background-color: #002244;
    }
    #mensajeRsvp {
      margin-top: 15px;
      padding: 10px;
      background-color: #e1f5e0;
      color: #2a6f33;
      border: 1px solid #2a6f33;
      display: none;
    }
    #errorRsvp {
      margin-top: 15px;
      padding: 10px;
      background-color: #fde2e2;
      color: #a52a2a;
      border: 1px solid #a52a2a;
      display: none;
    }
  </style>
</head>
<body>
  <h2>Confirmación de Asistencia</h2>

  <form id="formRsvp" onsubmit="return enviarRsvp();">
    <!-- 1. Desplegable con todos los invitados -->
    <label for="nombre">Selecciona tu nombre:</label>
    <select id="nombre" name="nombre" required>
      <option value="" disabled selected>— Elige tu nombre —</option>
      <!-- Aquí JavaScript insertará las opciones dinámicamente -->
    </select>

    <!-- 2. Radiobutton para confirmar Sí/No -->
    <label>¿Confirmas tu asistencia?</label>
    <input type="radio" id="conf_s" name="confirma" value="Sí" required>
    <label for="conf_s">Sí</label>
    <input type="radio" id="conf_n" name="confirma" value="No">
    <label for="conf_n">No</label>

    <!-- 3. Cantidad de invitados (se limitará luego por JS) -->
    <label for="cantidad">Cantidad de invitados:</label>
    <input type="number"
           id="cantidad"
           name="cantidad"
           min="0"
           value="0"
           required>
    <small id="infoMax" style="color: #555;">&nbsp;</small>

    <!-- 4. Comentarios adicionales -->
    <label for="comentarios">Comentarios:</label>
    <textarea id="comentarios" name="comentarios" rows="3"></textarea>

    <button type="submit">Enviar Confirmación</button>
  </form>

  <!-- Mensajes de éxito/error -->
  <div id="mensajeRsvp">¡Gracias! Tu respuesta ha sido registrada.</div>
  <div id="errorRsvp"></div>

  <script>
    // --------------------------------------------------------
    //  1) URL pública de tu Sheet “Asignaciones” publicada en CSV
    // --------------------------------------------------------
    const URL_ASIGNACIONES_CSV = 
      "https://docs.google.com/spreadsheets/d/11AWTUW2ELm3I547ShYpRtRT_FwODWx-RffWjXvdCMHw/pub?gid=0&single=true&output=csv";

    // --------------------------------------------------------
    //  2) URL de tu Apps Script (este proyecto), con /exec al final
    // --------------------------------------------------------
    const URL_APPS_SCRIPT = 
      "https://script.google.com/macros/s/AKfycbwHAuXhusqE3U-bbcxBTgMR7mH20V9zyvRgzDng98zUSkpaCtnLC-daC8Qy18Gj2tMQIA/exec";

    // --------------------------------------------------------
    //  Objeto que cargará { "Juan Pérez": 2, "María González": 1, ... }
    // --------------------------------------------------------
    let asignaciones = {};

    // 1) Al cargar la página, bajamos el CSV de asignaciones y lo parseamos
    window.addEventListener("DOMContentLoaded", function() {
      fetch(URL_ASIGNACIONES_CSV)
        .then(response => response.text())
        .then(csvText => {
          const lineas = csvText.trim().split("\n");
          // Saltamos la primera fila (cabeceras)
          for (let i = 1; i < lineas.length; i++) {
            const fila = lineas[i].split(",");
            const nombre = fila[0].trim();
            const maxInv = parseInt(fila[1].trim(), 10);
            if (nombre && !isNaN(maxInv)) {
              asignaciones[nombre] = maxInv;
            }
          }
          poblarSelectNombres();
        })
        .catch(err => {
          console.error("Error al bajar el CSV de asignaciones:", err);
          document.getElementById("errorRsvp").innerText =
            "No se pudo cargar la lista de invitados. Intenta más tarde.";
          document.getElementById("errorRsvp").style.display = "block";
        });
    });

    // 2) Función para poblar el <select id="nombre">
    function poblarSelectNombres() {
      const selectNombre = document.getElementById("nombre");
      Object.keys(asignaciones).forEach(nombre => {
        const opt = document.createElement("option");
        opt.value = nombre;
        opt.textContent = nombre;
        selectNombre.appendChild(opt);
      });
      // Cuando cambie la selección, ajustamos el máximo de invitados
      selectNombre.addEventListener("change", function() {
        ajustarMaximoAcompañantes(this.value);
      });
    }

    // 3) Ajusta el atributo “max” del campo cantidad según el nombre elegido
    function ajustarMaximoAcompañantes(nombreInvitado) {
      const inputCantidad = document.getElementById("cantidad");
      const infoMax = document.getElementById("infoMax");

      if (asignaciones.hasOwnProperty(nombreInvitado)) {
        const maxPermitido = asignaciones[nombreInvitado];
        inputCantidad.max = maxPermitido;
        if (parseInt(inputCantidad.value, 10) > maxPermitido) {
          inputCantidad.value = maxPermitido;
        }
        infoMax.innerText = `Máximo permitido: ${maxPermitido}`;
      } else {
        inputCantidad.removeAttribute("max");
        infoMax.innerText = "";
      }
    }

    // 4) Al enviar el formulario, validamos y mandamos por POST a Apps Script
    function enviarRsvp() {
      const nombre = document.getElementById("nombre").value;
      if (!nombre) {
        alert("Por favor, selecciona tu nombre.");
        return false;
      }

      const cantidadStr = document.getElementById("cantidad").value;
      const cantidadNum = parseInt(cantidadStr, 10);
      const maxPermitido = asignaciones[nombre];

      if (isNaN(cantidadNum) || cantidadNum < 0) {
        alert("Ingresa un número válido de acompañantes (0 o más).");
        return false;
      }
      if (cantidadNum > maxPermitido) {
        alert(`Solo puedes traer hasta ${maxPermitido} acompañante(s).`);
        return false;
      }

      const confirma = document.querySelector('input[name="confirma"]:checked');
      if (!confirma) {
        alert("Debes indicar si confirmas o no tu asistencia.");
        return false;
      }

      const datos = {
        nombre:       nombre,
        confirma:     confirma.value,
        cantidad_inv: cantidadNum,
        comentarios:  document.getElementById("comentarios").value.trim()
      };

      fetch(URL_APPS_SCRIPT, {
        method: "POST",
        mode:   "no-cors",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(datos)
      })
      .then(() => {
        document.getElementById("mensajeRsvp").style.display = "block";
        document.getElementById("errorRsvp").style.display = "none";
        document.getElementById("formRsvp").reset();
        document.getElementById("infoMax").innerText = "";
      })
      .catch(err => {
        document.getElementById("errorRsvp").innerText =
          "Hubo un problema al registrar tu respuesta. Intenta de nuevo más tarde.";
        document.getElementById("errorRsvp").style.display = "block";
        console.error("Error al enviar RSVP:", err);
      });

      return false;
    }
  </script>
</body>
</html>
