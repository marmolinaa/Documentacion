
## 2. Obtención de los datos
El código para la obtención de los datos se encuentran en:
> Archivo: escritorio/views/nuevo_proyecto.py

> Clase: CrearNuevoCorpus(View)

Para la creación de un nuevo corpus se hace uso de la clase view y a través de los metodos get y post se crea el objeto del formulario y se recupera para posteriormente guardar la información en la base de datos. A continuación se describen los tres métodos necesarios para su creación, recuperacíon y guardado.


**get(self, request, *args, **kwargs)**

Es una función que crea un objeto del formulario **CorpusForm** y lo pasa como parametro en el contexto del renderizado del template html del formulario.
```python
def get(self, request, *args, **kwargs):
        if not request.user.is_authenticated:
            return redirect('index')
        formulario = self.formulario()
        
        if self.kwargs['modalidad'] == 'True':
            banderaParalelo = True
        else:
            banderaParalelo = False

        context = {
            'form': formulario,
            'paralelo': banderaParalelo
        }
        setSessionProyectoActual(request,context)
        return render(request, self.template_name, context)
```
> **Archivo:** escritorio/views/nuevo_proyecto.py || **Clase:** CrearNuevoCorpus(View) 
|| **Función:** get(self, request, *args, **kwargs)


**post(self, request, *args, **kwargs)**

Es una función que valida y recupera el objeto del formulario **CorpusForm** y lo pasa como parametro a la función ***guardar_corpus*** a través del método ***cleaned_data***.
```python
def post(self, request, *args, **kwargs):
        if not request.user.is_authenticated:
            return redirect('index')
        formulario = self.formulario(request.POST)

        if self.kwargs['modalidad'] == 'True':
            banderaParalelo = True
        else:
            banderaParalelo = False
        
        if formulario.is_valid():
            nombreProyecto = list(Proyecto.objects.filter(nombre=formulario.cleaned_data['formularioNombre']))
            if not nombreProyecto:
                self.guardar_corpus(formulario.cleaned_data, request.user, banderaParalelo, request.POST)
                if banderaParalelo == True:
                    return redirect('escritorio-corpus-paralelos')
                else:
                    return redirect('escritorio-corpus-no-paralelos')
            else:
                messages.error(request, 'Ocurrió un error al crear el corpus, el nombre del corpus no puede ser repetido')

        context = {
            'form': formulario,
            'paralelo': banderaParalelo
        }
        setSessionProyectoActual(request,context)
        return render(request, self.template_name, context)
```
> **Archivo:** escritorio/views/nuevo_proyecto.py || **Clase:** CrearNuevoCorpus(View) 
>  || **Función:** post(self, request, *args, **kwargs)


**guardar_corpus(self, valorFormulario, usuario, banderaParalelo, requestPOST)**

Es una función que guarda la información del formulario llenado pora crear un nuevo corpus. A través de diccionarios y su llave (***Formulario['key']***) se recupera la información de cada uno de los campos del formulario y se crea un objeto (Agregar un registro a una tabla) de la base de datos que guardará la información del nuevo corpus. Para guardar cada uno de los metadatos se crean ciclos for que iteran y crean un objeto correspondiente al métadato que se está creando.
```python
def guardar_corpus(self, valorFormulario, usuario, banderaParalelo, requestPOST):
        usuarioActual = UsuarioGeco.objects.get(usuario = usuario)
        
        #Creando directorio por proyecto
        nombreProyecto = valorFormulario['formularioNombre'].lower()
        nombreProyectoSinEspacios = nombreProyecto.replace(" ", "")

        #Limpiando caracteres de acéntos, diéresis, caracter ñ, etc
        nombreProyectoLimpio = unicodedata.normalize("NFKD", nombreProyectoSinEspacios).encode("ascii","ignore").decode("ascii")

        rutaCompletaProyecto = settings.MEDIA_ROOT + "/" + nombreProyectoLimpio
        repositorio = nombreProyectoLimpio
        #Para el caso de proyectos con el mismo nombre + 1 espacio
        try:
            os.mkdir(rutaCompletaProyecto, mode=0o777)
        except:
            posfijo = format(randint(0,10000),'04d')
            rutaCompletaProyecto = rutaCompletaProyecto + posfijo
            os.mkdir(rutaCompletaProyecto, mode=0o777)
            repositorio = nombreProyectoLimpio + posfijo

        nuevoProyecto = Proyecto(
            nombre = valorFormulario['formularioNombre'],
            descripcion = valorFormulario['formularioDescripcion'],
            cita = valorFormulario['formularioCita'],
            activo = True, 
            es_publico = valorFormulario['formularioVisibilidad'],
            es_colaborativo =  valorFormulario['formularioColaboradores'],
            es_paralelo = banderaParalelo,
            repositorio = repositorio
        )
        nuevoProyecto.save()
        
        Colaborador_proyecto.objects.create(
            colaborador_id=usuarioActual.id,
            proyecto_id=nuevoProyecto.id,
            tipo_colaborador_id=1,
            activo=1, vigencia=0,
            fecha_fin_vigencia= "2040-08-19T19:28:20.407Z",
            fecha_ultimo_ingreso= datetime.now(tz=timezone.utc)
        )

        # LOS METADATOS OBLIGATORIOS POR DEFECTO
        Proyecto_metadato.objects.create(
            proyecto_id = nuevoProyecto.id, 
            metadato_id=1, 
            es_obligatorio= 1
        )
        Proyecto_metadato.objects.create(
            proyecto_id = nuevoProyecto.id,
            metadato_id=2,
            es_obligatorio= 1
        )

        todosLosMetadatos = valorFormulario['formularioMetadatos']
        metadatosObligatorios = valorFormulario['formularioMetadatosObligatorios']

        for i in range(len(todosLosMetadatos)):
            metadato = Metadato.objects.get(metadato = todosLosMetadatos[i])
            if todosLosMetadatos[i] in metadatosObligatorios:
                # LOS METADATOS OBLIGATORIOS SELECCIONADOS POR EL USUARIO
                Proyecto_metadato.objects.create(
                    proyecto_id = nuevoProyecto.id,
                    metadato_id = metadato.id,
                    es_obligatorio= 1
                )
            else:
                # LOS METADATOS NO OBLIGATORIOS
                Proyecto_metadato.objects.create(
                    proyecto_id = nuevoProyecto.id,
                    metadato_id = metadato.id,
                    es_obligatorio= 0
                )
        
        todosLosPersonalizados = requestPOST.getlist('formularioMetadatosPersonalizados')
        personalizadosObligatorios = requestPOST.getlist('banderaMetaPersonalizados')

        for metadatoPersonalizado in todosLosPersonalizados:
            metadatoPersonalizadoObj = Metadato.objects.create(
                metadato=metadatoPersonalizado,
                es_general=0,
                es_catalogo=0,
                es_obligatorio_general=0,
                es_multimedia = 0
            )
        
            if metadatoPersonalizado in personalizadosObligatorios:
                # LOS METADATOS OBLIGATORIOS SELECCIONADOS POR EL USUARIO
                Proyecto_metadato.objects.create(
                    proyecto_id = nuevoProyecto.id,
                    metadato_id = metadatoPersonalizadoObj.id,
                    es_obligatorio= 1
                )
            else:
                # LOS METADATOS NO OBLIGATORIOS
                Proyecto_metadato.objects.create(
                    proyecto_id = nuevoProyecto.id,
                    metadato_id = metadatoPersonalizadoObj.id,
                    es_obligatorio= 0
                )
        
        # Guardando todos los archivos asociados a un documento, pueden ser achivos multimedia u ptrps archivos texto
        todosLosArchivosAsociados = requestPOST.getlist('formularioArchivosAsociados')
        ArchivosObligatorios = requestPOST.getlist('banderaArchivosAsociados')

        for metadatoArchivo in todosLosArchivosAsociados:
            metadatoArchivoSinEspacios = metadatoArchivo.replace(" ", "_")
            llave = 'banderaTextoAudio'+ metadatoArchivoSinEspacios
            banderaTexto = requestPOST[llave]
            metadatoArchivoObj = Metadato.objects.create(
                metadato=metadatoArchivo,
                es_general=0,
                es_catalogo=0,
                es_obligatorio_general=0,
                es_multimedia = 1,
                es_texto = int(banderaTexto),
                es_audio = not(int(banderaTexto))
            )

            if metadatoArchivo in ArchivosObligatorios:
                # LOS METADATOS OBLIGATORIOS SELECCIONADOS POR EL USUARIO
                Proyecto_metadato.objects.create(
                    proyecto_id = nuevoProyecto.id,
                    metadato_id = metadatoArchivoObj.id,
                    es_obligatorio= 1
                )
            else:
                # LOS METADATOS NO OBLIGATORIOS
                Proyecto_metadato.objects.create(
                    proyecto_id = nuevoProyecto.id,
                    metadato_id = metadatoArchivoObj.id,
                    es_obligatorio= 0
                )

        registroBitacora = Bitacora.objects.create(
            fecha = datetime.now(tz=timezone.utc),
            actividad = str("Creó el corpus: " + nuevoProyecto.nombre),
            usuario = usuarioActual,
            proyecto_id = nuevoProyecto.id
        )
```
> **Archivo:** escritorio/views/nuevo_proyecto.py || **Clase:** CrearNuevoCorpus(View) 
>  || **Función:** guardar_corpus(self, valorFormulario, usuario, banderaParalelo, requestPOST)
