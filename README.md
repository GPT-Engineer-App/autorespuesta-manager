# autorespuesta-manager

<?php
require_once 'database.php';

if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    $action = $_POST['action'];

    if ($action == 'add_autorespuesta') {
        $pregunta = $_POST['pregunta'] ?? null;
        $respuesta = $_POST['respuesta'];
        $parent_id = $_POST['parent_id'] ?? null;
        $nivel = $_POST['nivel'] ?? 0;

        $resultado = createAutoRespuesta($conn, $pregunta, $respuesta, $parent_id, $nivel);

        if ($resultado === true) {
            header('Location: botonbot.php');
            exit();
        } else {
            echo "Error al añadir la respuesta: $resultado";
        }
    } elseif ($action == 'delete_autorespuesta') {
        $autorespuesta_id = $_POST['autorespuesta_id'] ?? null;
        if ($autorespuesta_id) {
            deleteAutoRespuesta($conn, $autorespuesta_id);
            header('Location: botonbot.php');
            exit();
        }
}
}
$autorespuestas = getAutoRespuestas($conn);
$todasPreguntas = getAllQuestions($conn);

function imprimirAutoRespuestas($autorespuestas, $todasPreguntas, $nivel = 0) {
    echo "<div class='nivel-$nivel'>";

    foreach ($autorespuestas as $autorespuesta) {
        $colorClass = 'nivel-color-' . $nivel % 10;
        echo "<div class='card mb-3 $colorClass' style='margin-left: " . (30 * $nivel) . "px;'>";
        echo "<div class='card-body'>";
        echo "<h5 class='card-title'>" . htmlspecialchars($autorespuesta['pregunta'] ?? '') . "</h5>";
        if (!empty($autorespuesta['respuesta'])) {
            echo "<p class='card-text'>" . htmlspecialchars($autorespuesta['respuesta']) . "</p>";
        }

        // Formulario para añadir subrespuesta
        echo "<form action='botonbot.php' method='POST' class='d-flex align-items-center'>";
        echo "<input type='hidden' name='action' value='add_autorespuesta'>";
        echo "<input type='hidden' name='parent_id' value='" . $autorespuesta['id'] . "'>";
        echo "<input type='hidden' name='nivel' value='" . ($nivel + 1) . "'>";
        echo "<input type='text' class='form-control me-2' name='respuesta' placeholder='Añade otra respuesta' required>";
        echo "<button class='btn btn-primary' type='submit'><i class='bi bi-plus-lg'></i></button>";
        echo "</form>";

        // Formulario para eliminar respuesta
        echo "<form action='botonbot.php' method='POST'>";
        echo "<input type='hidden' name='action' value='delete_autorespuesta'>";
        echo "<input type='hidden' name='autorespuesta_id' value='" . $autorespuesta['id'] . "'>";
        echo "<button class='btn btn-danger' type='submit'><i class='bi bi-dash-lg'></i></button>";
        echo "</form>";

        // Desplegable para seleccionar pregunta a anidar
        echo "<div class='mt-3'>";
        echo "<select class='form-select' name='parent_id'>";
        echo "<option value=''>Selecciona una pregunta para anidar la respuesta...</option>";
        foreach ($todasPreguntas as $pregunta) {
            echo "<option value='{$pregunta['id']}'>" . htmlspecialchars($pregunta['pregunta']) . "</option>";
        }
        echo "</select>";
        echo "</div>";

        if (!empty($autorespuesta['subrespuestas'])) {
            // Asegúrate de pasar $todasPreguntas en las llamadas recursivas
            imprimirAutoRespuestas($autorespuesta['subrespuestas'], $todasPreguntas, $nivel + 1);
        }

        echo "</div></div>"; // Cierra card-body y card
    }

    echo "</div>"; // Cierra nivel-$nivel
}


// Función para obtener todas las preguntas
function getAllQuestions($conn) {
    $sql = "SELECT id, pregunta FROM autorespuestas WHERE pregunta + 1"; // Ajusta según tu esquema de BD y lógica
    $result = $conn->query($sql);
    $preguntas = [];
    if ($result && $result->num_rows > 0) {
        while ($row = $result->fetch_assoc()) {
            $preguntas[] = $row;
        }
    }
    return $preguntas;
}

?>
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Gestión de Autorespuestas</title>
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.1.3/dist/css/bootstrap.min.css" rel="stylesheet">
    <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bootstrap-icons/font/bootstrap-icons.css">
    <link rel="stylesheet" href="/styles.css">
</head>
<body>
    <div class="container py-5">
        <h1 class="mb-5 text-center">Gestión de Autorespuestas</h1>
        <div class="row justify-content-center">
            <div class="col-lg-8">
                <div class="card card-nivel-0">
                    <div class="card-header">Añadir Autorespuesta Principal</div>
                    <div class="card-body">
                        <form action="botonbot.php" method="POST">
                            <input type="hidden" name="action" value="add_autorespuesta">
                            <div class="mb-3">
                                <select class="form-select" name="tipo" id="tipoSelect" onchange="toggleFields();">
                                    <option value="pregunta">Pregunta Nueva</option>
                                    <option value="respuesta">Respuesta a Pregunta Existente</option>
                                </select>
                            </div>
                            <div id="preguntaField" class="mb-3">
                                <label for="pregunta" class="form-label">Pregunta</label>
                                <input type="text" class="form-control" name="pregunta" placeholder="Introduce la pregunta">
                            </div>
                            <div id="respuestaField" class="mb-3" style="display: none;">
                                <label for="respuesta" class="form-label">Respuesta</label>
                                <input type="text" class="form-control" name="respuesta" placeholder="Introduce la respuesta">
                                
                                <select class="form-select mt-3" name="parent_id" id="parentSelectField" style="display: none;">
                                <option value="">Selecciona una pregunta...</option>
                                <?php
                                // Suponiendo que tienes una función que te devuelve todas las preguntas disponibles
                                $preguntas = getAutoRespuestas($conn); // Asegúrate de que esta función devuelva todas las preguntas principales
                                foreach ($preguntas as $pregunta) {
                                    echo "<option value='" . htmlspecialchars($pregunta['id']) . "'>" . htmlspecialchars($pregunta['pregunta']) . "</option>";
                                }
                                ?>
                            </select>
                            </div>
                            <button type="submit" class="btn btn-primary mt-3">Añadir</button>
                        </form>
                    </div>
                </div>
                <!-- Aquí se imprimirán las autorespuestas -->
                <div class="list-group mt-4">
                    <?php imprimirAutoRespuestas($autorespuestas, 0); ?>
                </div>
            </div>
        </div>
    </div>
    <script>
        function toggleFields() {
            var tipo = document.getElementById('tipoSelect').value;
            var preguntaField = document.getElementById('preguntaField');
            var respuestaField = document.getElementById('respuestaField');
            var parentSelectField = document.getElementById('parentSelectField');
            preguntaField.style.display = tipo === 'pregunta' ? 'block' : 'none';
            respuestaField.style.display = tipo === 'respuesta' ? 'block' : 'none';
            parentSelectField.style.display = tipo === 'respuesta' ? 'block' : 'none';
        }
    </script>
    <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.1.3/dist/js/bootstrap.bundle.min.js"></script>
</body>
</html>
mejora la idea y vista de ese codigo 

## Collaborate with GPT Engineer

This is a [gptengineer.app](https://gptengineer.app)-synced repository 🌟🤖

Changes made via gptengineer.app will be committed to this repo.

If you clone this repo and push changes, you will have them reflected in the GPT Engineer UI.

## Setup

```sh
git clone https://github.com/GPT-Engineer-App/autorespuesta-manager.git
cd autorespuesta-manager
npm i
```

```sh
npm run dev
```

This will run a dev server with auto reloading and an instant preview.

## Tech stack

- [Vite](https://vitejs.dev/)
- [React](https://react.dev/)
- [Chakra UI](https://chakra-ui.com/)

## Requirements

- Node.js & npm - [install with nvm](https://github.com/nvm-sh/nvm#installing-and-updating)
