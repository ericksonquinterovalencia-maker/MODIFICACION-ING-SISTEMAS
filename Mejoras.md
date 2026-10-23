MEJORAS DEL CODIGO Andrade

1. Proteger las credenciales de la base de datos

Archivo: `.env`

CI_ENVIRONMENT = development

app.baseURL = 'http://localhost/eysfwis/public/'

database.default.hostname = localhost
database.default.database = eysfwis
database.default.username = ${DB_USERNAME}
database.default.password = ${DB_PASSWORD}
database.default.DBDriver = MySQLi
database.default.port = 3306
```

En el archivo `.env` real, configura las credenciales correspondientes a tu entorno y no las publiques en GitHub. Si tu versión de CodeIgniter no admite la sustitución de variables de esa forma, define la conexión en `app/Config/Database.php` mediante variables de entorno de PHP.



2. Restringir el acceso CORS

namespace App\Filters;

use CodeIgniter\Filters\FilterInterface;
use CodeIgniter\HTTP\RequestInterface;
use CodeIgniter\HTTP\ResponseInterface;

class ApiCors implements FilterInterface
{
    public function before(
        RequestInterface $request,
        $arguments = null
    ) {
        $origin = $request->getHeaderLine('Origin');

        $allowedOrigins = [
            'http://localhost:3000',
            'http://localhost:8080',
        ];

        if ($origin !== '' && in_array($origin, $allowedOrigins, true)) {
            service('response')
                ->setHeader('Access-Control-Allow-Origin', $origin)
                ->setHeader('Vary', 'Origin')
                ->setHeader(
                    'Access-Control-Allow-Methods',
                    'GET, POST, PUT, DELETE, OPTIONS'
                )
                ->setHeader(
                    'Access-Control-Allow-Headers',
                    'Content-Type, Authorization'
                );
        }

        if ($request->getMethod() === 'OPTIONS') {
            return service('response')->setStatusCode(204);
        }

        return null;
    }

    public function after(
        RequestInterface $request,
        ResponseInterface $response,
        $arguments = null
    ) {
        // No se necesita ejecutar ninguna acción posterior.
    }
}


3. Usar transacciones al registrar un correo

Archivo: `app/Controllers/Api/CorreoApiController.php`

En el método `create()`, reemplaze el bloque que inserta el correo por una operación transaccional. Conserva la validación y la comprobación de que existe la persona antes de este bloque.

$db = \Config\Database::connect();

$correoText = trim($input['correo']);
$fecha = $input['fechaoptencion'] ?? date('Y-m-d');
$tipo = trim($input['tipo'] ?? 'Personal');

try {
    $db->transBegin();

    $id = $this->correoModel->insert([
        'idpersona'       => $idpersona,
        'correo'          => $correoText,
        'nombre'          => $correoText,
        'tipo'             => $tipo,
        'fechaoptencion'  => $fecha,
    ]);

    if (!$id) {
        throw new \RuntimeException(
            'No se pudo registrar el correo.'
        );
    }

    // Mantén esta inserción únicamente si la tabla "correo"
    // es necesaria en tu esquema actual.
    if ($db->tableExists('correo')) {
        $ok = $db->table('correo')->insert([
            'idcorreo'          => $id,
            'idcorreoelectronico'=> $id,
            'idpersona'          => $idpersona,
            'correo'             => $correoText,
            'nombre'             => $correoText,
            'tipo'               => $tipo,
            'fechaoptencion'     => $fecha,
        ]);

        if (!$ok) {
            throw new \RuntimeException(
                'No se pudo sincronizar la tabla correo.'
            );
        }
    }

    if ($db->transStatus() === false) {
        throw new \RuntimeException(
            'La transacción ha fallado.'
        );
    }

    $db->transCommit();

    return $this->respondCreated([
        'status'  => 201,
        'message' => 'Correo registrado exitosamente',
        'data'    => $this->correoModel->getCorreoDetail($id),
    ]);
} catch (\Throwable $e) {
    $db->transRollback();

    log_message('error', 'Error al registrar correo: {error}', [
        'error' => $e->getMessage(),
    ]);

    return $this->failServerError(
        'No se pudo completar el registro.'
    );
}

Esto presupone que las tablas usan un motor transaccional, como InnoDB. Si ambas tablas representan el mismo dato, conviene revisar el esquema y eliminar la duplicación en lugar de mantener dos registros redundantes.


4. Validar correctamente los datos recibidos

En el método `create()` del mismo controlador, antes de insertar el correo, usa validaciones como estas:

$input = $this->request->getJSON(true);

if (!is_array($input)) {
    $input = $this->request->getPost();
}

$rules = [
    'idpersona' => 'required|is_natural_no_zero',
    'correo'    => 'required|valid_email|max_length[150]',
];

if (!$this->validateData($input, $rules)) {
    return $this->failValidationErrors(
        $this->validator->getErrors()
    );
}

$idpersona = (int) $input['idpersona'];
$correoText = strtolower(trim($input['correo']));

if (!$this->personaModel->find($idpersona)) {
    return $this->failNotFound(
        'La persona indicada no existe.'
    );
}

$fecha = $input['fechaoptencion'] ?? date('Y-m-d');

$date = \DateTimeImmutable::createFromFormat('!Y-m-d', $fecha);

if (!$date || $date->format('Y-m-d') !== $fecha) {
    return $this->failValidationErrors([
        'fechaoptencion' => 'La fecha debe tener el formato YYYY-MM-DD.',
    ]);
}

Así evito identificadores inválidos, correos mal formados y fechas inexistentes. 


5. Añadir paginación a la consulta de correos

Archivo: `app/Controllers/Api/CorreoApiController.php`

public function index($idpersonaParam = null)
{
    $page = max(1, (int) $this->request->getGet('page'));
    $limit = (int) ($this->request->getGet('limit') ?? 20);
    $limit = max(1, min($limit, 100));

    $search = $this->request->getGet('q')
        ?? $this->request->getGet('search');

    $idpersona = $idpersonaParam
        ?? $this->request->getGet('idpersona');

    // El método del modelo debe aplicar LIMIT y OFFSET
    // en la consulta SQL; no basta con paginar el array.
    $data = $this->correoModel->getCorreosPaginated(
        $search,
        $idpersona ? (int) $idpersona : null,
        $limit,
        ($page - 1) * $limit
    );

    return $this->respond([
        'status' => 200,
        'data'   => $data['items'],
        'pagination' => [
            'page'        => $page,
            'limit'       => $limit,
            'total'       => $data['total'],
            'total_pages' => (int) ceil($data['total'] / $limit),
        ],
    ]);
}
```

Para que funcione, agregue al modelo `CorreoElectronicoModel` un método `getCorreosPaginated()` que realice la consulta paginada. 

public function getCorreosPaginated(
    ?string $search,
    ?int $idpersona,
    int $limit,
    int $offset
): array {
    $builder = $this->builder();

    if ($idpersona !== null) {
        $builder->where('idpersona', $idpersona);
    }

    if ($search !== null && trim($search) !== '') {
        $builder->like('correo', trim($search));
    }

    $total = (clone $builder)->countAllResults();

    $items = $builder
        ->orderBy($this->primaryKey, 'DESC')
        ->get($limit, $offset)
        ->getResultArray();

    return [
        'items' => $items,
        'total' => $total,
    ];
}


6. Estandarizar las respuestas JSON

Crea `app/Controllers/Api/BaseApiController.php`:

namespace App\Controllers\Api;

use CodeIgniter\RESTful\ResourceController;

abstract class BaseApiController extends ResourceController
{
    protected $format = 'json';

    protected function successResponse(
        array $data,
        string $message = 'Operación exitosa',
        int $status = 200
    ) {
        return $this->respond([
            'success' => true,
            'message' => $message,
            'data'    => $data,
        ], $status);
    }

    protected function errorResponse(
        string $message,
        int $status = 400,
        array $errors = []
    ) {
        $response = [
            'success' => false,
            'message' => $message,
        ];

        if ($errors !== []) {
            $response['errors'] = $errors;
        }

        return $this->respond($response, $status);
    }
}
Después, los controladores `PersonaController`, `CorreoApiController`, `SexoApiController` y `EstadoCivilApiController` pueden extender `BaseApiController` en lugar de `ResourceController`.


7. Centralizar las operaciones y evitar duplicación

En `CorreoApiController.php`, un método privado para normalizar los datos antes de insertar o actualizar:

private function normalizeCorreo(array $input): array
{
    $correo = strtolower(trim($input['correo'] ?? ''));

    $data = [
        'correo' => $correo,
        'nombre' => $correo,
    ];

    if (isset($input['idpersona'])) {
        $data['idpersona'] = (int) $input['idpersona'];
    }

    if (isset($input['tipo'])) {
        $data['tipo'] = trim($input['tipo']);
    }

    if (isset($input['fechaoptencion'])) {
        $data['fechaoptencion'] = $input['fechaoptencion'];
    }

    return $data;
}

En el método `update()`, después de validar los datos:

$updateData = $this->normalizeCorreo($input);

if (isset($updateData['idpersona'])) {
    if (!$this->personaModel->find($updateData['idpersona'])) {
        return $this->failNotFound(
            'La persona indicada no existe.'
        );
    }
}

if (!$this->correoModel->update($id, $updateData)) {
    return $this->failServerError(
        'No se pudo actualizar el correo.'
    );
}

return $this->respond([
    'success' => true,
    'message' => 'Correo actualizado correctamente.',
]);

Esto centraliza la normalización, pero no elimina por sí solo la duplicación entre las tablas `correo` y `correo_electronico`. Para ello hay que confirmar las relaciones y las claves del esquema SQL.


8. Proteger los endpoints mediante autenticación

Como ejemplo inicial, implemento un filtro de API key. No sustituye a un sistema completo de usuarios y permisos, pero evita dejar las operaciones de escritura abiertas.

Archivo: `app/Filters/ApiKeyAuth.php`

namespace App\Filters;

use CodeIgniter\Filters\FilterInterface;
use CodeIgniter\HTTP\RequestInterface;
use CodeIgniter\HTTP\ResponseInterface;

class ApiKeyAuth implements FilterInterface
{
    public function before(
        RequestInterface $request,
        $arguments = null
    ) {
        $expectedKey = getenv('API_ACCESS_KEY');
        $providedKey = $request->getHeaderLine('X-API-Key');

        if (
            !$expectedKey ||
            !$providedKey ||
            !hash_equals($expectedKey, $providedKey)
        ) {
            return service('response')
                ->setStatusCode(401)
                ->setJSON([
                    'success' => false,
                    'message' => 'No autorizado.',
                ]);
        }

        return null;
    }

    public function after(
        RequestInterface $request,
        ResponseInterface $response,
        $arguments = null
    ) {
        // Sin acciones posteriores.
    }
}
```

 `API_ACCESS_KEY` como variable de entorno del servidor, no dentro del repositorio.

En `app/Config/Filters.php`, registro el alias:

public array $aliases = [
    // Conserva aquí los demás filtros existentes.
    'apiKeyAuth' => \App\Filters\ApiKeyAuth::class,
];


9. Crear pruebas automatizadas

Archivo: `tests/app/Controllers/Api/CorreoApiControllerTest.php`

Ejemplo de prueba de validación usando las herramientas de CodeIgniter 4:

namespace Tests\App\Controllers\Api;

use CodeIgniter\Test\CIUnitTestCase;
use CodeIgniter\Test\FeatureTestTrait;

class CorreoApiControllerTest extends CIUnitTestCase
{
    use FeatureTestTrait;

    public function testCorreoRechazaDatosInvalidos(): void
    {
        $result = $this->withBodyFormat('json')
            ->post('/api/correo', [
                'idpersona' => -5,
                'correo' => 'esto-no-es-un-correo',
            ]);

        $result->assertStatus(400);
    }

    public function testCorreoInexistenteDevuelve404(): void
    {
        $result = $this->get('/api/correo/999999');

        $result->assertStatus(404);
    }
}

Los códigos HTTP exactos dependen de cómo estén implementados los métodos `failValidationErrors()` y `failNotFound()` en CodeIgniter.


10. Documentar la API con OpenAPI

Crea `docs/openapi.yaml`:

openapi: 3.0.3

info:
  title: API EYSFWIS
  version: 1.0.0
  description: API REST para administrar personas y correos.

servers:
  - url: http://localhost/eysfwis/public/api

paths:
  /correo:
    get:
      summary: Listar correos
      parameters:
        - name: page
          in: query
          schema:
            type: integer
            minimum: 1
            default: 1
        - name: limit
          in: query
          schema:
            type: integer
            minimum: 1
            maximum: 100
            default: 20
      responses:
        "200":
          description: Lista de correos obtenida correctamente.

    post:
      summary: Registrar correo
      security:
        - ApiKeyAuth: []
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required:
                - idpersona
                - correo
              properties:
                idpersona:
                  type: integer
                  minimum: 1
                correo:
                  type: string
                  format: email
                  maxLength: 150
                tipo:
                  type: string
                  example: Personal
                fechaoptencion:
                  type: string
                  format: date
      responses:
        "201":
          description: Correo registrado.
        "400":
          description: Datos inválidos.
        "401":
          description: No autorizado.

  /correo/{id}:
    get:
      summary: Consultar correo por ID
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: integer
      responses:
        "200":
          description: Correo encontrado.
        "404":
          description: Correo no encontrado.

components:
  securitySchemes:
    ApiKeyAuth:
      type: apiKey
      in: header
      name: X-API-Key

Este archivo documenta las rutas de ejemplo.