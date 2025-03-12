<script lang="ts">
    import { onMount } from 'svelte';
    import { writable, type Writable } from 'svelte/store';
    import Chart from 'chart.js/auto';
    
    interface Oficio {
        id_oficio: number;
        numero_oficio: string;
        fecha_recepcion: string;
        tipo: string;
        interno: boolean;
        destinatario: string;
        cargo_destinatario: string;
        remitente: string;
        cargo_remitente: string;
        area: string;
        categoria: string;
        asunto: string;
        entidad: string;
        sucursal: string | null;
        termino: boolean;
        fecha_termino: string | null;
        atencion: string[];
        asignacion: string[];
        numero_respuestas: number | null;
        fecha_primera_respuesta: string | null;
        estatus: string;
    }
    
    interface DashboardData {
        vigentes: number;
        porVencer: number;
        vencidos: number;
        conocimiento: number;
        atendidos: number;
        atendidosFueraTiempo: number;
        totalOficios: number;
        porArea: Record<string, number>;
        porCategoria: Record<string, number>;
        porFecha: Record<string, number>;
    }
    
    // Campos disponibles para selección
    interface CampoSeleccionable {
        id: string;
        nombre: string;
        seleccionado: boolean;
    }
    
    let oficios: Writable<Oficio[]> = writable([]);
    let dashboardData: Writable<DashboardData> = writable({
        vigentes: 0,
        porVencer: 0,
        vencidos: 0,
        conocimiento: 0,
        atendidos: 0,
        atendidosFueraTiempo: 0,
        totalOficios: 0,
        porArea: {},
        porCategoria: {},
        porFecha: {}
    });
    let error: Writable<string | null> = writable(null);
    let fechaInicio = '2025-01-01';
    let fechaFin = new Date().toISOString().split('T')[0]; // Obtiene la fecha actual en formato YYYY-MM-DD
    
    // Lista de campos disponibles para la tabla
    let camposDisponibles: Writable<CampoSeleccionable[]> = writable([
        { id: "area", nombre: "Área", seleccionado: true },
        { id: "categoria", nombre: "Categoría", seleccionado: true },
        { id: "estatus", nombre: "Estatus", seleccionado: true },
        { id: "remitente", nombre: "Remitente", seleccionado: true },
        { id: "tipo", nombre: "Tipo", seleccionado: true },
        { id: "destinatario", nombre: "Destinatario", seleccionado: false },
        { id: "asunto", nombre: "Asunto", seleccionado: false },
        { id: "entidad", nombre: "Entidad", seleccionado: false }
    ]);
    
    // Tipo de fecha a filtrar
    let recepcionOTermino: boolean = true; // true = fecha_recepcion, false = fecha_termino
    
    let barChart: Chart | undefined;
    let pieChartEstatus: Chart | undefined;
    let barChartUAR: Chart | undefined;
    let pieChartNaturaleza: Chart | undefined;
    let lineChart: Chart | undefined;
    let loading = false;
    
    // Función para convertir fecha YYYY-MM-DD a timestamp Unix en milisegundos
    const fechaATimestamp = (fechaString: string): number => {
        // Asegura que la fecha tenga zona horaria al inicio del día
        const fecha = new Date(fechaString + 'T00:00:00');
        return fecha.getTime();
    };
    
    // Función para determinar el estatus de un oficio - CORREGIDA
    const calcularEstatus = (
        fechaTermino: string | null,
        fechaPrimeraRespuesta: string | null,
        termino: boolean
    ): string => {
        // Si no hay fecha de término o el oficio no requiere término, es de conocimiento
        if (!fechaTermino || !termino) {
            return "CONOCIMIENTO";
        }
        
        const fechaTerminoDate = new Date(fechaTermino);
        const hoy = new Date();
        
        // Si hay fecha de primera respuesta, verificar si fue atendido en tiempo o fuera de tiempo
        if (fechaPrimeraRespuesta) {
            const fechaRespuestaDate = new Date(fechaPrimeraRespuesta);
            if (fechaRespuestaDate <= fechaTerminoDate) {
                return "ATENDIDO EN TIEMPO";
            } else {
                return "ATENDIDO FUERA DE TIEMPO";
            }
        }
        
        // Si no hay respuesta aún, verificar el estado según la fecha de término
        // Cálculo de días de diferencia (considerando día completo)
        const diffDias = Math.ceil((fechaTerminoDate.getTime() - hoy.getTime()) / (1000 * 60 * 60 * 24));
        
        if (diffDias <= 0) {
            return "VENCIDO";
        } else if (diffDias <= 3) { // 3 días o menos
            return "POR VENCER";
        } else {
            return "VIGENTE";
        }
    };
    
    // Procesar los datos para el dashboard - MEJORADO
    const procesarDatosParaDashboard = (datos: Oficio[]): DashboardData => {
        const resultado: DashboardData = {
            vigentes: 0,
            porVencer: 0,
            vencidos: 0,
            conocimiento: 0,
            atendidos: 0,
            atendidosFueraTiempo: 0,
            totalOficios: datos.length,
            porArea: {},
            porCategoria: {},
            porFecha: {}
        };
        
        // Procesar cada oficio
        for (const oficio of datos) {
            // Calcular el estatus real usando la función mejorada
            const estatus = calcularEstatus(
                oficio.fecha_termino,
                oficio.fecha_primera_respuesta,
                oficio.termino
            );
            
            // Actualizar el estatus en el objeto oficio para uso posterior si es necesario
            oficio.estatus = estatus;
            
            // Incrementar contador según estatus
            switch (estatus) {
                case "VIGENTE":
                    resultado.vigentes++;
                    break;
                case "POR VENCER":
                    resultado.porVencer++;
                    break;
                case "VENCIDO":
                    resultado.vencidos++;
                    break;
                case "CONOCIMIENTO":
                    resultado.conocimiento++;
                    break;
                case "ATENDIDO EN TIEMPO":
                    resultado.atendidos++;
                    break;
                case "ATENDIDO FUERA DE TIEMPO":
                    resultado.atendidosFueraTiempo++;
                    break;
            }
            
            // Conteo por área (asegurarse de que exista un valor)
            const area = oficio.area || "SIN ASIGNAR";
            resultado.porArea[area] = (resultado.porArea[area] || 0) + 1;
            
            // Conteo por categoría (asegurarse de que exista un valor)
            const categoria = oficio.categoria || "SIN CATEGORÍA";
            resultado.porCategoria[categoria] = (resultado.porCategoria[categoria] || 0) + 1;
            
            // Datos históricos por fecha de recepción
            if (oficio.fecha_recepcion) {
                const fechaFormateada = new Date(oficio.fecha_recepcion).toLocaleDateString('es-MX');
                resultado.porFecha[fechaFormateada] = (resultado.porFecha[fechaFormateada] || 0) + 1;
            }
        }
        
        return resultado;
    };
    
    // Obtener los campos seleccionados para la tabla
    const obtenerCamposSeleccionados = (): string[] => {
        let camposSeleccionados: string[] = [];
        
        camposDisponibles.update(campos => {
            camposSeleccionados = campos.filter(campo => campo.seleccionado).map(campo => campo.id);
            return campos;
        });
        
        return camposSeleccionados;
    };
    
    // Función fetchData mejorada
    const fetchData = async () => {
        try {
            loading = true;
            error.set(null);
            
            // Convertir fechas a timestamp de Unix en milisegundos
            const fechaInicioTimestamp = fechaATimestamp(fechaInicio);
            const fechaFinTimestamp = fechaATimestamp(fechaFin);
            
            // Obtener campos seleccionados para la tabla
            const camposTabla = obtenerCamposSeleccionados();
            
            console.log(`Enviando fechas: inicio=${fechaInicio} (${fechaInicioTimestamp}), fin=${fechaFin} (${fechaFinTimestamp})`);
            console.log('Campos seleccionados:', camposTabla);
            console.log('Tipo de fecha:', recepcionOTermino ? 'Recepción' : 'Término');
            
            const response = await fetch('/api/tablero', {
                method: 'POST',
                headers: { 
                    'Content-Type': 'application/json',
                    'Accept': 'application/json'
                },
                body: JSON.stringify({
                    fecha_inicio: fechaInicioTimestamp,
                    fecha_fin: fechaFinTimestamp,
                    campos_tabla: camposTabla,
                    recepcion_o_termino: recepcionOTermino
                })
            });
            
            if (!response.ok) {
                let errorMessage;
                try {
                    const errorData = await response.json();
                    console.error(`Error HTTP ${response.status}:`, errorData);
                    errorMessage = `Error del servidor: ${response.status} ${errorData.error || response.statusText}`;
                    if (errorData.fullError) {
                        console.error('Error completo:', errorData.fullError);
                    }
                } catch (e) {
                    // Si no se puede parsear como JSON, leer como texto
                    const errorText = await response.text();
                    console.error(`Error HTTP ${response.status}:`, errorText);
                    errorMessage = `Error del servidor: ${response.status} ${response.statusText}`;
                }
                throw new Error(errorMessage);
            }
            
            // Verificar el tipo de contenido
            const contentType = response.headers.get('content-type');
            if (!contentType || !contentType.includes('application/json')) {
                console.warn('La respuesta no es JSON. Contenido:', await response.text());
                throw new Error('El servidor no devolvió datos en formato JSON');
            }
            
            const rawData = await response.json();
            console.log("Datos recibidos:", rawData);
            
            // Manejar tanto un objeto único como un array de oficios
            const oficiosArray = Array.isArray(rawData) ? rawData : [rawData];
            
            // Almacenar datos brutos
            oficios.set(oficiosArray);
            
            // Procesar datos para el dashboard
            const datosProcesados = procesarDatosParaDashboard(oficiosArray);
            dashboardData.set(datosProcesados);
            
            // Actualizar gráficos
            actualizarGraficos(datosProcesados);
        } catch (err: any) {
            console.error("Error en fetchData:", err);
            error.set(err.message || 'Error al obtener datos');
            
            // Cargar datos de ejemplo
            cargarDatosEjemplo();
        } finally {
            loading = false;
        }
    };
    
    // Función para cargar datos de ejemplo si el API falla
    const cargarDatosEjemplo = () => {
        const datosEjemplo: DashboardData = {
            vigentes: 25,
            porVencer: 10,
            vencidos: 5,
            conocimiento: 8,
            atendidos: 12,
            atendidosFueraTiempo: 3,
            totalOficios: 63,
            porArea: {
                "ORGANO INTERNO DE CONTROL ESPECIFICO": 15,
                "DIRECCION GENERAL": 8,
                "UNIDAD DE TRANSPARENCIA": 12,
                "SUBDIRECCIÓN JURÍDICA": 5
            },
            porCategoria: {
                "RESPONSABILIDADES": 7,
                "INFORMACIÓN": 18,
                "CONSULTA": 10,
                "INSTRUCCIÓN": 5
            },
            porFecha: {
                "01/03/2025": 3,
                "02/03/2025": 5,
                "03/03/2025": 2,
                "04/03/2025": 4,
                "05/03/2025": 6,
                "06/03/2025": 3,
                "07/03/2025": 7
            }
        };
        
        dashboardData.set(datosEjemplo);
        actualizarGraficos(datosEjemplo);
    };
    
    // Función para seleccionar o deseleccionar todos los campos
    const toggleTodosCampos = (seleccionar: boolean) => {
        camposDisponibles.update(campos => {
            return campos.map(campo => ({
                ...campo,
                seleccionado: seleccionar
            }));
        });
    };
    
    const actualizarGraficos = (data: DashboardData) => {
        try {
            // Gráfico de barras principal - Estatus de oficios
            const barCanvas = document.getElementById('barChart') as HTMLCanvasElement;
            if (barCanvas) {
                const barCtx = barCanvas.getContext('2d');
                if (barCtx && barChart) barChart.destroy();
                if (barCtx) {
                    barChart = new Chart(barCtx, {
                        type: 'bar',
                        data: {
                            labels: ['Vigente', 'Por vencer', 'Vencidos', 'Conocimiento', 'Atendidos', 'Fuera de tiempo'],
                            datasets: [{
                                label: 'Oficios',
                                data: [
                                    data.vigentes,
                                    data.porVencer,
                                    data.vencidos,
                                    data.conocimiento,
                                    data.atendidos,
                                    data.atendidosFueraTiempo
                                ],
                                backgroundColor: [
                                    'hsl(120,70%,45%)', // verde
                                    'hsl(60,70%,45%)',  // amarillo
                                    'hsl(0,70%,45%)',   // rojo
                                    'gray',             // gris para conocimiento
                                    'blue',             // azul para atendidos en tiempo
                                    'purple'            // morado para atendidos fuera de tiempo
                                ]
                            }]
                        },
                        options: {
                            responsive: true,
                            plugins: {
                                legend: {
                                    display: false
                                }
                            }
                        }
                    });
                }
            }
        
            // Gráfico de pie - Distribución por estatus
            const pieCanvasEstatus = document.getElementById('pieChartEstatus') as HTMLCanvasElement;
            if (pieCanvasEstatus) {
                const pieCtxEstatus = pieCanvasEstatus.getContext('2d');
                if (pieCtxEstatus && pieChartEstatus) pieChartEstatus.destroy();
                if (pieCtxEstatus) {
                    pieChartEstatus = new Chart(pieCtxEstatus, {
                        type: 'pie',
                        data: {
                            labels: ['Vigente', 'Por vencer', 'Vencidos', 'Conocimiento', 'Atendidos', 'Fuera de tiempo'],
                            datasets: [{
                                data: [
                                    data.vigentes,
                                    data.porVencer,
                                    data.vencidos,
                                    data.conocimiento,
                                    data.atendidos,
                                    data.atendidosFueraTiempo
                                ],
                                backgroundColor: [
                                    'hsl(120,70%,45%)', // verde
                                    'hsl(60,70%,45%)',  // amarillo
                                    'hsl(0,70%,45%)',   // rojo
                                    'gray',             // gris para conocimiento
                                    'blue',             // azul para atendidos en tiempo
                                    'purple'            // morado para atendidos fuera de tiempo
                                ]
                            }]
                        },
                        options: {
                            responsive: true
                        }
                    });
                }
            }
        
            // Gráfico de barras - Distribución por área
            if (Object.keys(data.porArea).length > 0) {
                const barCanvasUAR = document.getElementById('barChartUAR') as HTMLCanvasElement;
                if (barCanvasUAR) {
                    const barCtxUAR = barCanvasUAR.getContext('2d');
                    if (barCtxUAR && barChartUAR) barChartUAR.destroy();
                    if (barCtxUAR) {
                        // Limitamos a las 5 áreas con más oficios si hay más de 5
                        let areas = Object.entries(data.porArea)
                            .sort((a, b) => b[1] - a[1])
                            .slice(0, 5);
                        
                        barChartUAR = new Chart(barCtxUAR, {
                            type: 'bar',
                            data: {
                                labels: areas.map(item => item[0]),
                                datasets: [{
                                    label: 'Oficios por Área',
                                    data: areas.map(item => item[1]),
                                    backgroundColor: [
                                        'rgba(54, 162, 235, 0.8)',
                                        'rgba(75, 192, 192, 0.8)',
                                        'rgba(153, 102, 255, 0.8)',
                                        'rgba(255, 159, 64, 0.8)',
                                        'rgba(255, 99, 132, 0.8)'
                                    ]
                                }]
                            },
                            options: {
                                responsive: true,
                                indexAxis: 'y',  // Barras horizontales para mejor legibilidad
                                plugins: {
                                    legend: {
                                        display: false
                                    }
                                }
                            }
                        });
                    }
                }
            }
        
            // Gráfico de pie - Distribución por categoría
            if (Object.keys(data.porCategoria).length > 0) {
                const pieCanvasNaturaleza = document.getElementById('pieChartNaturaleza') as HTMLCanvasElement;
                if (pieCanvasNaturaleza) {
                    const pieCtxNaturaleza = pieCanvasNaturaleza.getContext('2d');
                    if (pieCtxNaturaleza && pieChartNaturaleza) pieChartNaturaleza.destroy();
                    if (pieCtxNaturaleza) {
                        pieChartNaturaleza = new Chart(pieCtxNaturaleza, {
                            type: 'pie',
                            data: {
                                labels: Object.keys(data.porCategoria),
                                datasets: [{
                                    data: Object.values(data.porCategoria),
                                    backgroundColor: [
                                        'rgba(255, 99, 132, 0.7)',
                                        'rgba(54, 162, 235, 0.7)',
                                        'rgba(255, 206, 86, 0.7)',
                                        'rgba(75, 192, 192, 0.7)',
                                        'rgba(153, 102, 255, 0.7)',
                                        'rgba(255, 159, 64, 0.7)'
                                    ]
                                }]
                            },
                            options: {
                                responsive: true
                            }
                        });
                    }
                }
            }
        
            // Gráfico de línea - Tendencia por fecha
            if (Object.keys(data.porFecha).length > 0) {
                const lineCanvas = document.getElementById('lineChart') as HTMLCanvasElement;
                if (lineCanvas) {
                    const lineCtx = lineCanvas.getContext('2d');
                    if (lineCtx && lineChart) lineChart.destroy();
                    if (lineCtx) {
                        // Ordenar fechas cronológicamente
                        const fechasOrdenadas = Object.entries(data.porFecha)
                            .sort((a, b) => {
                                const fechaA = new Date(a[0].split('/').reverse().join('-'));
                                const fechaB = new Date(b[0].split('/').reverse().join('-'));
                                return fechaA.getTime() - fechaB.getTime();
                            });
                        
                        // Tomar solo los últimos 15 días si hay muchos datos
                        const ultimosDias = fechasOrdenadas.slice(-15);
                        
                        lineChart = new Chart(lineCtx, {
                            type: 'line',
                            data: {
                                labels: ultimosDias.map(item => item[0]),
                                datasets: [{
                                    label: 'Oficios Recibidos',
                                    data: ultimosDias.map(item => item[1]),
                                    borderColor: 'rgba(54, 162, 235, 1)',
                                    backgroundColor: 'rgba(54, 162, 235, 0.2)',
                                    fill: true,
                                    tension: 0.1
                                }]
                            },
                            options: {
                                responsive: true
                            }
                        });
                    }
                }
            }
        } catch (err) {
            console.error("Error al actualizar gráficos:", err);
        }
    };
    
    onMount(() => {
        // Asegurar que el DOM esté listo antes de hacer la solicitud inicial
        setTimeout(fetchData, 100);
    });
</script>
        
    <div class="p-4">
        <div class="flex flex-col sm:flex-row gap-4 mb-4">
            <div class="flex items-center gap-2">
                <label for="fecha-inicio">Desde:</label>
                <input id="fecha-inicio" type="date" bind:value={fechaInicio} class="border p-2 rounded" />
            </div>
            <div class="flex items-center gap-2">
                <label for="fecha-fin">Hasta:</label>
                <input id="fecha-fin" type="date" bind:value={fechaFin} class="border p-2 rounded" />
            </div>
            <div class="flex items-center gap-2">
                <label for="tipo-fecha">Tipo de fecha:</label>
                <select id="tipo-fecha" bind:value={recepcionOTermino} class="border p-2 rounded">
                    <option value={true}>Fecha de recepción</option>
                    <option value={false}>Fecha de término</option>
                </select>
            </div>
        </div>
        
        <!-- Selección de campos -->
        <div class="mb-4 border p-4 rounded-lg bg-gray-50">
            <h3 class="font-bold mb-2">Selección de campos para la consulta</h3>
            <div class="mb-2 flex gap-2">
                <button 
                    on:click={() => toggleTodosCampos(true)} 
                    class="bg-blue-500 text-white p-1 rounded text-sm"
                >
                    Seleccionar todos
                </button>
                <button 
                    on:click={() => toggleTodosCampos(false)} 
                    class="bg-gray-500 text-white p-1 rounded text-sm"
                >
                    Deseleccionar todos
                </button>
            </div>
            <div class="grid grid-cols-2 sm:grid-cols-4 gap-2">
                {#each $camposDisponibles as campo (campo.id)}
                    <div class="flex items-center gap-1">
                        <input 
                            type="checkbox" 
                            id={campo.id} 
                            bind:checked={campo.seleccionado} 
                            class="form-checkbox h-4 w-4 text-blue-600"
                        />
                        <label for={campo.id} class="text-sm">{campo.nombre}</label>
                    </div>
                {/each}
            </div>
        </div>
        
        <div class="mb-4 flex justify-center">
            <button 
                on:click={fetchData} 
                class="bg-blue-500 text-white p-2 rounded flex items-center justify-center w-full sm:w-auto sm:px-4"
                disabled={loading}
            >
                {#if loading}
                    <span class="mr-2">Actualizando...</span>
                {:else}
                    <span>Actualizar Tablero</span>
                {/if}
            </button>
        </div>
        
        {#if $error}
        <div class="bg-red-100 border border-red-400 text-red-700 px-4 py-3 rounded mb-4" role="alert">
            <p class="font-bold">Error</p>
            <p>{$error}</p>
            <p class="mt-2 text-sm">Se están mostrando datos de ejemplo.</p>
        </div>
        {/if}
    </div>
        
    <div class="grid grid-cols-1 sm:grid-cols-3 gap-4 p-4">
        <div class="bg-green-500 text-white p-4 rounded-lg shadow-lg text-center">
            <h2 class="text-xl font-bold">Vigentes</h2>
            <p class="text-3xl">{$dashboardData.vigentes}</p>
        </div>
        <div class="bg-yellow-500 text-white p-4 rounded-lg shadow-lg text-center">
            <h2 class="text-xl font-bold">Por vencer</h2>
            <p class="text-3xl">{$dashboardData.porVencer}</p>
        </div>
        <div class="bg-red-500 text-white p-4 rounded-lg shadow-lg text-center">
            <h2 class="text-xl font-bold">Vencidos</h2>
            <p class="text-3xl">{$dashboardData.vencidos}</p>
        </div>
    </div>
    
    <div class="grid grid-cols-1 sm:grid-cols-2 gap-4 p-4">
        <div class="bg-gray-100 p-4 rounded-lg shadow-lg text-center">
            <h2 class="text-xl font-bold">Conocimiento</h2>
            <p class="text-3xl">{$dashboardData.conocimiento}</p>
        </div>
        <div class="bg-blue-500 text-white p-4 rounded-lg shadow-lg text-center">
            <h2 class="text-xl font-bold">Atendidos en tiempo</h2>
            <p class="text-3xl">{$dashboardData.atendidos}</p>
        </div>
    </div>
        
    <div class="p-4">
        <h3 class="text-center font-bold">Resumen de Oficios por Estatus</h3>
        <canvas id="barChart" width="400" height="200"></canvas>
    </div>
        
    <div class="grid grid-cols-1 sm:grid-cols-3 gap-4 p-4">
        <div class="p-2 border rounded">
            <h3 class="text-center font-bold">Distribución por Estatus</h3>
            <canvas id="pieChartEstatus" width="200" height="200"></canvas>
        </div>
        <div class="p-2 border rounded">
            <h3 class="text-center font-bold">Distribución por Área</h3>
            <canvas id="barChartUAR" width="200" height="200"></canvas>
        </div>
        <div class="p-2 border rounded">
            <h3 class="text-center font-bold">Distribución por Categoría</h3>
            <canvas id="pieChartNaturaleza" width="200" height="200"></canvas>
        </div>
    </div>
        
    <div class="p-4">
        <h3 class="text-center font-bold">Total de Oficios Recibidos (Últimos 15 días)</h3>
        <canvas id="lineChart" width="400" height="200"></canvas>
    </div>
    
    <div class="p-4">
        <h3 class="text-center font-bold">Total de oficios: {$dashboardData.totalOficios}</h3>
    </div>
