# Sistemas-operativos-grupo10
int main() {
    srand(static_cast<unsigned>(time(nullptr)));

    sem_init(&sem_vacios, 0, TAM_BUFER); 
    sem_init(&sem_llenos, 0, 0);       

    pthread_t sensores[NUM_SENSORES];
    pthread_t modulos[NUM_MODULOS_ANALISIS];
    int ids_sensores[NUM_SENSORES];
    int ids_modulos[NUM_MODULOS_ANALISIS];

    tiempo_inicio_ms = marca_tiempo_ms();

    log_linea("=== SIGET: simulacion productor-consumidor ===");
    std::ostringstream cfg;
    cfg << "Sensores: " << NUM_SENSORES
        << "  |  Modulos de analisis: " << NUM_MODULOS_ANALISIS
        << "  |  Capacidad del bufer: " << TAM_BUFER
        << "  |  Lecturas por sensor: " << LECTURAS_POR_SENSOR;
    log_linea(cfg.str());
    log_linea("");
    imprimir_encabezado();

    for (int i = 0; i < NUM_MODULOS_ANALISIS; ++i) {
        ids_modulos[i] = i + 1;
        pthread_create(&modulos[i], nullptr, funcion_modulo_analisis, &ids_modulos[i]);
    }


    for (int i = 0; i < NUM_SENSORES; ++i) {
        ids_sensores[i] = i + 1;
        pthread_create(&sensores[i], nullptr, funcion_sensor, &ids_sensores[i]);
    }


    for (int i = 0; i < NUM_SENSORES; ++i) {
        pthread_join(sensores[i], nullptr);
    }

    log_linea("---------|-------------|--------------------------------|----------------------------");
    log_linea("[MAIN] Todos los sensores finalizaron. Enviando senales de apagado...");

 
    for (int i = 0; i < NUM_MODULOS_ANALISIS; ++i) {
        sem_wait(&sem_vacios);
        pthread_mutex_lock(&mutex_bufer);

        LecturaTrafico pildora{};
        pildora.es_pildora = true;
        bufer[indice_escritura] = pildora;
        indice_escritura = (indice_escritura + 1) % TAM_BUFER;

        pthread_mutex_unlock(&mutex_bufer);
        sem_post(&sem_llenos);
    }

    
    for (int i = 0; i < NUM_MODULOS_ANALISIS; ++i) {
        pthread_join(modulos[i], nullptr);
    }

    log_linea("---------|-------------|--------------------------------|-------------------------");
    log_linea("");
    log_linea("=== RESUMEN ===");
    std::ostringstream l1, l2, l3;
    l1 << "  Lecturas generadas    : " << total_generadas.load();
    l2 << "  Lecturas procesadas   : " << total_procesadas.load();
    l3 << "  Alertas de congestion : " << alertas_congestion.load();
    log_linea(l1.str());
    log_linea(l2.str());
    log_linea(l3.str());

    if (total_generadas.load() == total_procesadas.load()) {
        log_linea("  [VERIFICACION] OK: no se perdio ni se duplico ninguna lectura.");
    } else {
        log_linea("  [VERIFICACION] ERROR: hay una inconsistencia en el conteo.");
    }

    sem_destroy(&sem_vacios);
    sem_destroy(&sem_llenos);
    pthread_mutex_destroy(&mutex_bufer);
    pthread_mutex_destroy(&mutex_consola);
    return 0;
}
