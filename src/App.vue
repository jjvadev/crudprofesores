<template>
  <div class="shell">
    <!-- NAV / TOPBAR -->
    <header class="nav no-print">
      <div class="brand">
        <div class="logo">🎓</div>
        <div class="brand-text">
          <div class="title">Gestor de Profesores</div>
          <div class="subtitle">Vue 3 · LocalStorage</div>
        </div>
      </div>

      <div class="actions">
        <button class="btn primary" @click="openCreate">➕ Nuevo</button>

        <label class="btn" title="Importar JSON">
          ⬆️ Importar
          <input class="file" type="file" accept="application/json" @change="handleImport" />
        </label>

        <button class="btn" @click="exportJSON" title="Exportar JSON">⬇️ Exportar</button>
        <button class="btn" @click="printAll" title="Imprimir lista">🖨️ Imprimir</button>
        <button class="btn danger" @click="resetDB" title="Borrar datos locales">🗑️ Reset</button>
      </div>
    </header>

    <!-- FILTER BAR -->
    <section class="filters no-print">
      <div class="field wide">
        <label>Cédula (Enter abre la ficha)</label>
        <div class="row">
          <input v-model.trim="quickCedula" @keyup.enter="goQuickCedula" placeholder="Ingresa cédula y presiona Enter" />
          <button class="btn" @click="goQuickCedula">Ir</button>
        </div>
      </div>
      <div class="field">
        <label>Buscar por cédula o nombre</label>
        <input v-model.trim="query" placeholder="Ej: 1012345678 o Ana" />
      </div>
      <div class="field">
        <label>Tipo</label>
        <select v-model="filtroTipo">
          <option value="">Todos</option>
          <option value="Planta">Planta</option>
          <option value="Catedrático">Catedrático</option>
          <option value="Contrato">Contrato</option>
        </select>
      </div>
      <div class="field compact">
        <label>Vista</label>
        <div class="segmented">
          <button :class="{on: view==='cards'}" @click="view='cards'">Tarjetas</button>
          <button :class="{on: view==='table'}" @click="view='table'">Tabla</button>
          <button :class="{on: view==='semana'}" @click="view='semana'">Semana</button>
        </div>
      </div>
      <div class="counter">
        <span class="chip">{{ filtrados.length }} resultado(s)</span>
      </div>
    </section>

    <!-- LIST — CARDS -->
    <section v-if="view==='cards'" class="cards">
      <div v-if="filtrados.length===0" class="empty">
        Sin resultados. Agrega profesores o ajusta los filtros.
      </div>

      <article v-for="p in filtrados" :key="p.cedula" class="card">
        <div class="card-head">
          <div class="person">
            <div class="avatar" aria-hidden>👤</div>
            <div>
              <div class="name">{{ p.nombre }}</div>
              <div class="id mono">{{ p.cedula }}</div>
            </div>
          </div>
          <span :class="['pill', typePill(p.tipo)]">{{ p.tipo }}</span>
        </div>

        <div class="grid">
          <div>
            <div class="muted">Horas semanales</div>
            <div>{{ p.horasSemanales ?? '—' }}</div>
          </div>
          <div>
            <div class="muted">Correo</div>
            <div>{{ p.email || '—' }}</div>
          </div>
          <div>
            <div class="muted">Ubicación de trabajo</div>
            <div>{{ p.ubicacionTrabajo || '—' }}</div>
          </div>
          <div>
            <div class="muted">Residencia</div>
            <div>{{ p.lugarResidencia || '—' }}</div>
          </div>
        </div>

        <!-- Chips de actividades si hay estructura -->
        <div v-if="(p.actividadesLista && p.actividadesLista.length) || p.actividades" class="activities">
          <template v-if="p.actividadesLista && p.actividadesLista.length">
            <span v-for="(a,i) in p.actividadesLista" :key="i" class="act-chip" :title="a.nota || ''">
              {{ a.nombre }} <small v-if="a.horas">({{ a.horas }}h)</small>
            </span>
          </template>
          <template v-else>
            <span class="act-note">{{ p.actividades }}</span>
          </template>
        </div>

        <div class="card-actions">
          <button class="link" @click="verDetalle(p)">Ver</button>
          <button class="link" @click="openEdit(p)">Editar</button>
          <button class="link" @click="imprimirUno(p)">Imprimir</button>
          <button class="link danger" @click="removeTeacher(p)">Eliminar</button>
        </div>
      </article>
    </section>

    <!-- LIST — TABLE -->
    <section v-else-if="view==='table'" class="table-card">
      <div class="table-wrapper">
        <table class="table">
          <thead>
            <tr>
              <th>Cédula</th>
              <th>Nombre</th>
              <th>Tipo</th>
              <th>Horas/Sem</th>
              <th>Ubicación</th>
              <th>Residencia</th>
              <th class="text-right">Acciones</th>
            </tr>
          </thead>
          <tbody>
            <tr v-if="filtrados.length === 0">
              <td class="empty" colspan="7">Sin resultados. Agrega profesores o ajusta los filtros.</td>
            </tr>
            <tr v-for="p in filtrados" :key="p.cedula">
              <td class="mono">{{ p.cedula }}</td>
              <td class="semi">{{ p.nombre }}</td>
              <td><span :class="['pill', typePill(p.tipo)]">{{ p.tipo }}</span></td>
              <td>{{ p.horasSemanales ?? '—' }}</td>
              <td>{{ p.ubicacionTrabajo || '—' }}</td>
              <td>{{ p.lugarResidencia || '—' }}</td>
              <td>
                <div class="row-actions">
                  <button class="link" @click="verDetalle(p)">Ver</button>
                  <button class="link" @click="openEdit(p)">Editar</button>
                  <button class="link" @click="imprimirUno(p)">Imprimir</button>
                  <button class="link danger" @click="removeTeacher(p)">Eliminar</button>
                </div>
              </td>
            </tr>
          </tbody>
        </table>
      </div>
    </section>

    <!-- WEEK PLANNER -->
    <section v-else class="planner">
      <div class="planner-head">
        <div class="legend">
          <span class="legend-item planta">Planta</span>
          <span class="legend-item catedra">Catedrático</span>
          <span class="legend-item contrato">Contrato</span>
        </div>
        <div class="muted">Vista semanal de horarios ({{ startHour }}:00–{{ endHour }}:00)</div>
      </div>

      <div class="planner-grid">
        <div class="hours-rail">
          <div v-for="h in hourMarks" :key="h" class="hour">{{ pad(h) }}:00</div>
        </div>
        <div class="days" :style="{height: plannerHeight+'px'}">
          <div class="day-header" v-for="d in days" :key="d">{{ d }}</div>

          <div class="day-col" v-for="(d,di) in days" :key="'col-'+d" :style="{height: plannerHeight+'px'}">
            <div v-for="h in hourMarks" :key="d+'-'+h" class="hour-line"></div>

            <!-- bloques -->
            <div
              v-for="(b, i) in semanaBlocks.filter(x=>x.dayIndex===di)"
              :key="d+'-b-'+i"
              class="block"
              :class="b.kind"
              :style="{ top: b.top+'px', height: b.height+'px' }"
              :title="b.title+' — '+b.subtitle"
            >
              <div class="b-title">{{ b.title }}</div>
              <div class="b-sub">{{ b.subtitle }}</div>
            </div>
          </div>
        </div>
      </div>

      <div v-if="semanaBlocks.length===0" class="empty mt">No hay horarios en la vista actual.</div>
    </section>

    <!-- TOASTS -->
    <div class="toasts">
      <div v-for="t in toasts" :key="t.id" class="toast">{{ t.msg }}</div>
    </div>

    <!-- MODAL: FORM -->
    <div v-if="showForm" class="overlay">
      <div class="modal">
        <div class="modal-head">
          <div>
            <h2>{{ formMode === 'create' ? 'Nuevo profesor' : 'Editar profesor' }}</h2>
            <small class="muted">Obligatorios: <span class="req">Cédula</span>, <span class="req">Nombre</span> y <span class="req">Tipo</span>.</small>
          </div>
          <button class="icon-btn" @click="closeForm" aria-label="Cerrar">✖</button>
        </div>

        <form class="form-grid" @submit.prevent="save">
          <div class="field">
            <label>Cédula *</label>
            <input v-model.trim="form.cedula" :disabled="formMode==='edit'" required placeholder="1012345678" />
          </div>
          <div class="field">
            <label>Nombre completo *</label>
            <input v-model.trim="form.nombre" required placeholder="Ana María Pérez" />
          </div>
          <div class="field">
            <label>Tipo *</label>
            <select v-model="form.tipo" required>
              <option>Planta</option>
              <option>Catedrático</option>
              <option>Contrato</option>
            </select>
          </div>
          <div class="field">
            <label>Horas semanales</label>
            <input v-model.number="form.horasSemanales" type="number" min="0" placeholder="16" />
          </div>

          <!-- ACTIVIDADES ESTRUCTURADAS -->
          <div class="col-2">
            <div class="subhead">
              <label>Actividades (chips) — opcional</label>
              <button type="button" class="btn" @click="addActividad">Añadir actividad</button>
            </div>
            <div v-if="!form.actividadesLista.length" class="muted">Agrega actividades con nombre y horas semanales.</div>
            <div v-for="(a, ai) in form.actividadesLista" :key="'act-'+ai" class="actividad-row">
              <input v-model.trim="a.nombre" placeholder="Nombre (ej. Tutorías)" />
              <input v-model.number="a.horas" min="0" type="number" placeholder="Horas/sem" />
              <input v-model.trim="a.nota" placeholder="Nota (opcional)" />
              <button type="button" class="icon-btn" @click="form.actividadesLista.splice(ai,1)">🗑️</button>
            </div>
          </div>

          <!-- NOTAS LIBRES DE ACTIVIDADES -->
          <div class="field col-2">
            <label>Actividades (texto libre)</label>
            <textarea v-model.trim="form.actividades" rows="2" placeholder="Clases, investigación, tutorías…"></textarea>
          </div>

          <div class="field">
            <label>Ubicación de trabajo</label>
            <input v-model.trim="form.ubicacionTrabajo" placeholder="Bloque A - Of. 203" />
          </div>
          <div class="field">
            <label>Lugar de residencia</label>
            <input v-model.trim="form.lugarResidencia" placeholder="Barrio/Ciudad" />
          </div>
          <div class="field">
            <label>Teléfono</label>
            <input v-model.trim="form.telefono" placeholder="Opcional" />
          </div>
          <div class="field">
            <label>Correo</label>
            <input v-model.trim="form.email" type="email" placeholder="Opcional" />
          </div>

          <!-- HORARIOS -->
          <div class="col-2">
            <div class="subhead">
              <label>Horarios (opcional)</label>
              <div class="sub-actions">
                <button type="button" class="btn" @click="addHorario">Añadir horario</button>
                <button type="button" class="btn subtle" @click="addHorarioPlantilla('Lunes','08:00','10:00','Clase','A-101')">+ Plantilla</button>
              </div>
            </div>
            <div v-if="form.horarios.length===0" class="muted">Aún no has agregado horarios.</div>

            <div v-for="(h,idx) in form.horarios" :key="'h-'+idx" class="horario">
              <select v-model="h.dia">
                <option value="Lunes">Lunes</option>
                <option value="Martes">Martes</option>
                <option value="Miércoles">Miércoles</option>
                <option value="Jueves">Jueves</option>
                <option value="Viernes">Viernes</option>
                <option value="Sábado">Sábado</option>
                <option value="Domingo">Domingo</option>
              </select>
              <input v-model="h.inicio" type="time" />
              <input v-model="h.fin" type="time" />
              <input v-model.trim="h.materia" placeholder="Materia" />
              <div class="horario-right">
                <input v-model.trim="h.aula" placeholder="Aula" />
                <button type="button" class="icon-btn" @click="form.horarios.splice(idx,1)">🗑️</button>
              </div>
            </div>
            <div v-if="horarioWarnings.length" class="warnings">
              <div v-for="(w,i) in horarioWarnings" :key="'w-'+i">⚠️ {{ w }}</div>
            </div>
          </div>

          <div class="form-actions col-2">
            <button type="button" class="btn" @click="closeForm">Cancelar</button>
            <button type="submit" class="btn primary">Guardar</button>
          </div>
        </form>
      </div>
    </div>

    <!-- MODAL: DETAIL -->
    <div v-if="showDetail" class="overlay">
      <div class="modal">
        <div class="modal-head">
          <h2>Ficha de profesor</h2>
          <button class="icon-btn" @click="showDetail=false">✖</button>
        </div>

        <div v-if="detalle" class="detail print-card">
          <div class="detail-top">
            <div>
              <div class="muted">Cédula</div>
              <div class="mono">{{ detalle.cedula }}</div>
            </div>
            <span :class="['pill', typePill(detalle.tipo)]">{{ detalle.tipo }}</span>
          </div>

          <h3 class="detail-name">{{ detalle.nombre }}</h3>

          <div class="detail-grid">
            <div>
              <div class="muted">Horas semanales</div>
              <div>{{ detalle.horasSemanales ?? '—' }}</div>
            </div>
            <div>
              <div class="muted">Correo</div>
              <div>{{ detalle.email || '—' }}</div>
            </div>
            <div>
              <div class="muted">Ubicación de trabajo</div>
              <div>{{ detalle.ubicacionTrabajo || '—' }}</div>
            </div>
            <div>
              <div class="muted">Lugar de residencia</div>
              <div>{{ detalle.lugarResidencia || '—' }}</div>
            </div>

            <div class="col-2" v-if="detalle.actividadesLista && detalle.actividadesLista.length">
              <div class="muted">Actividades</div>
              <div class="act-chips">
                <span v-for="(a,i) in detalle.actividadesLista" :key="'da-'+i" class="act-chip" :title="a.nota || ''">
                  {{ a.nombre }} <small v-if="a.horas">({{ a.horas }}h)</small>
                </span>
              </div>
            </div>
            <div class="col-2" v-if="detalle.actividades && !detalle.actividadesLista?.length">
              <div class="muted">Actividades (nota)</div>
              <div class="pre">{{ detalle.actividades }}</div>
            </div>
          </div>

          <div class="mt">
            <div class="muted mb">Horarios</div>
            <div v-if="!detalle.horarios || detalle.horarios.length===0" class="muted">—</div>
            <div v-else class="table-wrapper">
              <table class="table compact">
                <thead>
                  <tr>
                    <th>Día</th>
                    <th>Inicio</th>
                    <th>Fin</th>
                    <th>Materia</th>
                    <th>Aula</th>
                  </tr>
                </thead>
                <tbody>
                  <tr v-for="(h,i) in detalle.horarios" :key="'dh-'+i">
                    <td>{{ h.dia }}</td>
                    <td>{{ h.inicio }}</td>
                    <td>{{ h.fin }}</td>
                    <td>{{ h.materia || '—' }}</td>
                    <td>{{ h.aula || '—' }}</td>
                  </tr>
                </tbody>
              </table>
            </div>
          </div>

          <div class="detail-actions no-print">
            <button class="btn" @click="imprimirUno(detalle)">Imprimir</button>
            <button class="btn" @click="showDetail=false">Cerrar</button>
          </div>
        </div>
      </div>
    </div>

  </div>
</template>

<script setup>
import { ref, computed, watch, onMounted } from 'vue'

const STORAGE_KEY = 'db_profesores_v1'

/* ===== State ===== */
const profesores = ref([])
const query = ref('')
const filtroTipo = ref('')
const quickCedula = ref('')
const view = ref('cards') // 'cards' | 'table' | 'semana'

const showForm = ref(false)
const showDetail = ref(false)
const detalle = ref(null)
const formMode = ref('create')
const form = ref(emptyForm())
const toasts = ref([])
const horarioWarnings = ref([])

/* ===== Constants for planner ===== */
const startHour = 6
const endHour = 22
const pxPerMin = 1 // 1px == 1min => 16h = 960px (suave al scrollear)
const plannerHeight = (endHour - startHour) * 60 * pxPerMin
const days = ['Lunes','Martes','Miércoles','Jueves','Viernes','Sábado','Domingo']
const hourMarks = Array.from({length: endHour - startHour + 1}, (_,i)=> i + startHour)

/* ===== Computed ===== */
const filtrados = computed(() => {
  const q = query.value.trim().toLowerCase()
  return profesores.value
    .filter(p => !filtroTipo.value || p.tipo === filtroTipo.value)
    .filter(p => !q ? true : String(p.cedula).toLowerCase().includes(q) || (p.nombre || '').toLowerCase().includes(q))
    .sort((a, b) => (a.nombre || '').localeCompare(b.nombre || ''))
})

const semanaBlocks = computed(() => {
  // construir bloques para la vista semanal desde los profesores filtrados
  const blocks = []
  for (const p of filtrados.value) {
    if (!Array.isArray(p.horarios)) continue
    for (const h of p.horarios) {
      if (!h || !h.dia || !h.inicio || !h.fin) continue
      const di = dayIndex(h.dia)
      if (di < 0) continue
      const s = toMinutes(h.inicio)
      const e = toMinutes(h.fin)
      if (s == null || e == null || e <= s) continue
      const top = (s - startHour * 60) * pxPerMin
      const height = (e - s) * pxPerMin
      blocks.push({
        dayIndex: di,
        top,
        height,
        kind: kindClass(p.tipo),
        title: (h.materia || 'Clase') + (h.aula ? ` · ${h.aula}` : ''),
        subtitle: `${p.nombre} (${p.tipo})`,
      })
    }
  }
  return blocks
})

/* ===== Lifecycle ===== */
onMounted(() => {
  loadDB()
  if (profesores.value.length === 0) {
    profesores.value = [
      { cedula: '1012345678', nombre: 'Ana María Pérez', tipo: 'Planta', horasSemanales: 16,
        actividadesLista: [{ nombre: 'Tutorías', horas: 2 }, { nombre: 'Investigación', horas: 4 }],
        actividades: 'Clases de Cálculo I y asesorías.',
        ubicacionTrabajo: 'Bloque A - Of. 203', lugarResidencia: 'Laureles, Medellín',
        telefono: '3001234567', email: 'ana.perez@uni.edu',
        horarios: [ { dia: 'Lunes', inicio: '08:00', fin: '10:00', materia: 'Cálculo I', aula: 'A-201' } ] },
      { cedula: '1098765432', nombre: 'Carlos Gómez', tipo: 'Catedrático', horasSemanales: 8,
        actividadesLista: [{ nombre: 'Cátedra', horas: 8 }],
        actividades: 'Cátedra de Programación.',
        ubicacionTrabajo: 'Bloque B - Lab. 1', lugarResidencia: 'Envigado',
        telefono: '', email: 'carlos.gomez@uni.edu', horarios: [] },
      { cedula: '1029384756', nombre: 'Luisa Fernández', tipo: 'Contrato', horasSemanales: 12,
        actividadesLista: [{ nombre: 'Proyecto', horas: 12 }],
        actividades: 'Proyecto de investigación.',
        ubicacionTrabajo: 'Bloque C - Of. 110', lugarResidencia: 'Bello',
        telefono: '', email: '', horarios: [] },
    ]
    saveDB()
    toast('Base de ejemplo cargada')
  }
})

/* ===== Watchers ===== */
watch(() => form.value.horarios, () => {
  horarioWarnings.value = validateHorarios(form.value.horarios)
}, { deep: true })

/* ===== Helpers & UI ===== */
function emptyForm () {
  return {
    cedula: '',
    nombre: '',
    tipo: 'Planta',
    horasSemanales: null,
    actividades: '',
    actividadesLista: [],
    ubicacionTrabajo: '',
    lugarResidencia: '',
    telefono: '',
    email: '',
    horarios: [],
  }
}
function toast (msg) {
  const id = Math.random().toString(36).slice(2)
  toasts.value.push({ id, msg })
  setTimeout(() => { toasts.value = toasts.value.filter(t => t.id !== id) }, 2200)
}
function typePill (tipo) {
  switch (tipo) {
    case 'Planta': return 'pill--planta'
    case 'Catedrático': return 'pill--catedra'
    case 'Contrato': return 'pill--contrato'
    default: return 'pill--neutral'
  }
}
function kindClass (tipo) {
  return tipo === 'Planta' ? 'k-planta' : tipo === 'Catedrático' ? 'k-catedra' : 'k-contrato'
}
function pad(n){ return String(n).padStart(2,'0') }
function dayIndex(d) {
  const map = { 'Lunes':0,'Martes':1,'Miércoles':2,'Miercoles':2,'Jueves':3,'Viernes':4,'Sábado':5,'Sabado':5,'Domingo':6 }
  return map[d] ?? -1
}
function toMinutes(t) {
  if (!t || !/^\d{2}:\d{2}$/.test(t)) return null
  const [hh, mm] = t.split(':').map(Number)
  return hh * 60 + mm
}

/* ===== DB ===== */
function loadDB () {
  try {
    const raw = localStorage.getItem(STORAGE_KEY)
    profesores.value = raw ? JSON.parse(raw) : []
  } catch (e) {
    console.error('Error cargando DB', e)
    profesores.value = []
  }
}
function saveDB () { localStorage.setItem(STORAGE_KEY, JSON.stringify(profesores.value)) }
function resetDB () {
  if (!confirm('Esto borrará todos los datos locales. ¿Continuar?')) return
  localStorage.removeItem(STORAGE_KEY)
  profesores.value = []
  toast('DB reiniciada')
}

/* ===== CRUD ===== */
function openCreate () {
  formMode.value = 'create'
  form.value = emptyForm()
  horarioWarnings.value = []
  showForm.value = true
}
function openEdit (p) {
  formMode.value = 'edit'
  form.value = JSON.parse(JSON.stringify(p))
  if (!Array.isArray(form.value.actividadesLista)) form.value.actividadesLista = []
  horarioWarnings.value = validateHorarios(form.value.horarios || [])
  showForm.value = true
}
function closeForm () {
  showForm.value = false
  form.value = emptyForm()
  horarioWarnings.value = []
}
function addHorario () {
  form.value.horarios.push({ dia: 'Lunes', inicio: '', fin: '', materia: '', aula: '' })
}
function addHorarioPlantilla(dia,inicio,fin,materia,aula){
  form.value.horarios.push({ dia, inicio, fin, materia, aula })
}
function addActividad(){
  form.value.actividadesLista.push({ nombre: '', horas: null, nota: '' })
}
function validateHorarios(list) {
  const warnings = []
  const groups = {}
  for (const h of list) {
    if (!h.dia) continue
    const s = toMinutes(h.inicio), e = toMinutes(h.fin)
    if (h.inicio && h.fin && (s == null || e == null)) warnings.push(`Formato inválido en ${h.dia} (${h.inicio}–${h.fin}).`)
    if (s != null && e != null && e <= s) warnings.push(`Hora fin debe ser mayor que inicio en ${h.dia} (${h.inicio}–${h.fin}).`)
    if (!groups[h.dia]) groups[h.dia] = []
    if (s != null && e != null) groups[h.dia].push([s,e,h])
  }
  for (const dia in groups) {
    const arr = groups[dia].sort((a,b)=>a[0]-b[0])
    for (let i=1;i<arr.length;i++){
      const prev = arr[i-1], cur = arr[i]
      if (cur[0] < prev[1]) {
        const p = prev[2], c = cur[2]
        warnings.push(`Solape en ${dia}: "${p.materia||'Bloque'}" (${mins(prev[0])}–${mins(prev[1])}) con "${c.materia||'Bloque'}" (${mins(cur[0])}–${mins(cur[1])}).`)
      }
    }
  }
  return warnings
}
function mins(m){ return `${pad(Math.floor(m/60))}:${pad(m%60)}` }

function save () {
  if (!form.value.cedula || !form.value.nombre || !form.value.tipo) {
    alert('Por favor completa CÉDULA, NOMBRE y TIPO.')
    return
  }
  // Horarios inválidos bloquean guardar
  const errs = validateHorarios(form.value.horarios || [])
  if (errs.length) {
    alert('Revisa los horarios:\n\n' + errs.join('\n'))
    return
  }

  form.value.cedula = String(form.value.cedula).trim()

  if (formMode.value === 'create') {
    const exists = profesores.value.some(p => String(p.cedula).trim() === form.value.cedula)
    if (exists) { alert('Ya existe un profesor con esa cédula.'); return }
    profesores.value.push(JSON.parse(JSON.stringify(form.value)))
    toast('Profesor creado')
  } else {
    const idx = profesores.value.findIndex(p => String(p.cedula).trim() === form.value.cedula)
    if (idx !== -1) profesores.value[idx] = JSON.parse(JSON.stringify(form.value))
    toast('Profesor actualizado')
  }
  saveDB()
  closeForm()
}
function removeTeacher (p) {
  if (!confirm(`¿Eliminar a ${p.nombre} (${p.cedula})?`)) return
  profesores.value = profesores.value.filter(x => x.cedula !== p.cedula)
  saveDB()
  toast('Profesor eliminado')
}
function verDetalle (p) {
  detalle.value = p
  showDetail.value = true
}

/* ===== Quick search by ID ===== */
function goQuickCedula () {
  if (!quickCedula.value) return
  const p = profesores.value.find(x => String(x.cedula).trim() === String(quickCedula.value).trim())
  if (p) { verDetalle(p); toast('Profesor encontrado') } else { toast('No existe esa cédula') }
}

/* ===== Print ===== */
function imprimirUno (p) {
  const html = renderFichaHTML(p)
  const w = window.open('', '_blank')
  w.document.write(html)
  w.document.close()
  w.focus()
  w.print()
}
function printAll () {
  if (filtrados.value.length === 0) { alert('No hay profesores para imprimir.'); return }
  const cards = filtrados.value.map(p => renderFichaInnerHTML(p)).join('')
  const html = `<!DOCTYPE html><html lang="es"><head><meta charset="utf-8"/><title>Profesores</title>
  <style>${printStyles()}</style></head><body><h1>Profesores</h1><div class="grid">${cards}</div></body></html>`
  const w = window.open('', '_blank')
  w.document.write(html)
  w.document.close()
  w.focus()
  w.print()
}
function renderFichaHTML (p) {
  const inner = renderFichaInnerHTML(p)
  return `<!DOCTYPE html><html lang="es"><head><meta charset="utf-8"/><title>Profesor ${p.nombre}</title>
  <style>${printStyles()}</style></head><body>${inner}</body></html>`
}
function printStyles () {
  return `body{font-family:Inter,system-ui,Arial;padding:24px;max-width:980px;margin:0 auto;background:#fff}
  h1{margin:0 0 12px;font-size:20px}
  .grid{display:grid;grid-template-columns:repeat(auto-fill,minmax(280px,1fr));gap:16px}
  .card{border:1px solid #e5e7eb;border-radius:12px;padding:16px}
  .badge{display:inline-block;border:1px solid #e5e7eb;border-radius:999px;padding:.125rem .5rem;font-size:12px;font-weight:600}
  .muted{color:#64748b;font-size:12px;margin-bottom:4px}
  table{width:100%;border-collapse:collapse} th,td{border:1px solid #e5e7eb;padding:6px;text-align:left} th{background:#f8fafc}`
}
function renderFichaInnerHTML (p) {
  const color = p.tipo === 'Planta' ? '#16a34a' : (p.tipo === 'Catedrático' ? '#d97706' : '#2563eb')
  const horarios = (p.horarios && p.horarios.length)
    ? `<div style="margin-top:12px">
        <div class="muted">Horarios</div>
        <table><thead><tr><th>Día</th><th>Inicio</th><th>Fin</th><th>Materia</th><th>Aula</th></tr></thead>
          <tbody>
            ${p.horarios.map(h => `<tr><td>${h.dia}</td><td>${h.inicio||'—'}</td><td>${h.fin||'—'}</td><td>${h.materia||'—'}</td><td>${h.aula||'—'}</td></tr>`).join('')}
          </tbody>
        </table>
      </div>` : ''
  // Actividades resumen
  const acts = Array.isArray(p.actividadesLista) && p.actividadesLista.length
    ? '<div class="muted">Actividades</div><ul>' + p.actividadesLista.map(a=>`<li>${a.nombre}${a.horas?` (${a.horas}h)`:''}${a.nota?` — ${a.nota}`:''}</li>`).join('') + '</ul>'
    : (p.actividades ? `<div class="muted">Actividades</div><div>${p.actividades.replace(/\n/g,'<br>')}</div>` : '')
  return `<div class="card">
    <div style="display:flex;justify-content:space-between;align-items:center;gap:8px">
      <div>
        <div class="muted">Cédula</div>
        <div style="font-family:ui-monospace,monospace;font-size:16px">${p.cedula}</div>
      </div>
      <span class="badge" style="color:${color}">${p.tipo}</span>
    </div>
    <h2 style="margin:6px 0 8px;font-size:18px">${p.nombre}</h2>
    <div style="display:grid;grid-template-columns:repeat(auto-fit,minmax(200px,1fr));gap:8px">
      <div><div class="muted">Horas semanales</div><div>${p.horasSemanales ?? '—'}</div></div>
      <div><div class="muted">Correo</div><div>${p.email||'—'}</div></div>
      <div><div class="muted">Ubicación de trabajo</div><div>${p.ubicacionTrabajo||'—'}</div></div>
      <div><div class="muted">Lugar de residencia</div><div>${p.lugarResidencia||'—'}</div></div>
      <div style="grid-column:1/-1">${acts}</div>
    </div>
    ${horarios}
  </div>`
}

/* ===== Import / Export ===== */
function exportJSON () {
  const data = JSON.stringify(profesores.value, null, 2)
  const blob = new Blob([data], { type: 'application/json' })
  const url = URL.createObjectURL(blob)
  const a = document.createElement('a')
  const date = new Date().toISOString().slice(0, 10)
  a.href = url
  a.download = `profesores_${date}.json`
  a.click()
  URL.revokeObjectURL(url)
  toast('JSON exportado')
}
function handleImport (e) {
  const file = e.target.files?.[0]
  if (!file) return
  const reader = new FileReader()
  reader.onload = () => {
    try {
      const incoming = JSON.parse(reader.result)
      if (!Array.isArray(incoming)) throw new Error('El JSON debe ser un arreglo de profesores')
      const map = new Map(profesores.value.map(p => [String(p.cedula).trim(), p]))
      for (const p of incoming) {
        if (!p || !p.cedula) continue
        const key = String(p.cedula).trim()
        map.set(key, {
          ...map.get(key),
          ...p,
          cedula: key,
          horarios: Array.isArray(p.horarios) ? p.horarios : [],
          actividadesLista: Array.isArray(p.actividadesLista) ? p.actividadesLista : (map.get(key)?.actividadesLista || []),
        })
      }
      profesores.value = Array.from(map.values())
      saveDB()
      toast('Importación completada')
    } catch (err) {
      console.error(err)
      alert('No se pudo importar el archivo. Verifica el formato JSON.')
    }
    e.target.value = ''
  }
  reader.readAsText(file)
}
</script>

<style scoped>
/* ===== THEME: Soft & Solid (sin transparencias, UX/UI claro) ===== */
:root{
  --bg: #f4f6f9;            /* fondo suave */
  --panel: #ffffff;         /* panel SOLIDO */
  --panel-2: #fbfcfe;
  --text: #263241;          /* gris oscuro legible (no negro puro) */
  --muted: #6b7280;
  --border: #e6eaf0;
  --border-strong: #cfd6e0;
  --accent: #3b82f6;        /* azul amable */
  --accent-600: #2563eb;
  --ok: #16a34a;
  --warn: #d97706;
  --danger: #dc2626;

  --radius-s: 10px;
  --radius-m: 14px;
  --radius-l: 16px;

  --shadow-s: 0 1px 2px rgba(38,50,65,.06);
  --shadow-m: 0 6px 18px rgba(38,50,65,.10);
  --shadow-l: 0 16px 36px rgba(38,50,65,.14);
}

/* Base */
* { box-sizing: border-box }
html { font-size: 16px }
body, .shell { background: var(--bg); color: var(--text); -webkit-font-smoothing: antialiased; -moz-osx-font-smoothing: grayscale }
h1,h2,h3 { margin: 0; line-height: 1.25; font-weight: 800; letter-spacing: .2px }
p, label, input, select, textarea, button, table { font-size: 1rem; line-height: 1.5 }
a, button { font-weight: 600 }
.mono { font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, 'Liberation Mono', 'Courier New', monospace }
.semi { font-weight: 700 }
.muted { color: var(--muted) }
.text-right { text-align: right }
.req { color: var(--danger) }
.mb { margin-bottom: .5rem }
.mt { margin-top: 1rem }

/* Layout general */
.shell { min-height: 100vh; display: grid; grid-template-rows: auto auto 1fr; gap: 12px; padding-bottom: 16px }

/* Topbar (sólido) */
.nav{
  display:flex; align-items:center; justify-content:space-between; gap:16px;
  padding: 14px 18px; background: var(--panel); border: 1px solid var(--border);
  border-radius: var(--radius-l); box-shadow: var(--shadow-m); margin: 14px 14px 0 14px;
}
.brand{ display:flex; align-items:center; gap:12px }
.logo{
  width:44px; height:44px; display:grid; place-items:center; border-radius: 12px;
  background: linear-gradient(135deg, #93c5fd, var(--accent));
  color: #0f172a; font-weight: 900; box-shadow: var(--shadow-s);
}
.title{ font-weight: 900; letter-spacing:.2px; font-size: 1.1rem }
.subtitle{ font-size:.8rem; color: var(--muted) }
.brand-text{ display:grid; gap:2px }

.actions{ display:flex; gap:10px; flex-wrap:wrap; align-items:center }

/* Botones */
.btn{
  border: 1.5px solid var(--border);
  background: var(--panel);
  color: var(--text);
  padding: 10px 14px;
  border-radius: var(--radius-s);
  cursor: pointer;
  transition: .15s;
  box-shadow: var(--shadow-s);
}
.btn:hover{ box-shadow: var(--shadow-m); transform: translateY(-1px) }
.btn:focus-visible{ outline: 3px solid #bfdbfe; outline-offset: 2px }
.btn.primary{
  color:#fff; background: var(--accent);
  border-color: var(--accent);
}
.btn.primary:hover{ background: var(--accent-600) }
.btn.danger{ color:#fff; background: var(--danger); border-color: var(--danger) }
.icon-btn{
  border: 1.5px solid var(--border);
  background: var(--panel);
  color: var(--text);
  padding: 6px 10px;
  border-radius: var(--radius-s);
  cursor: pointer;
}
.link{
  background: transparent; border: 0; color: var(--accent); cursor: pointer;
  padding: 6px 8px; border-radius: 8px;
}
.link:hover{ background: #e8f0ff }
.link.danger{ color: var(--danger) }

label.btn { position: relative; overflow: hidden }
label.btn input[type="file"]{ position:absolute; inset:0; opacity:0; cursor:pointer }

/* Filtros (sólido) */
.filters{
  display:grid; grid-template-columns: 2fr 2fr 1.2fr auto auto; gap:12px;
  margin: 0 14px; padding: 14px 18px; background: var(--panel);
  border: 1px solid var(--border); border-radius: var(--radius-l); box-shadow: var(--shadow-m);
  align-items: end;
}
.field{ display:grid; gap:6px }
.field.wide{ grid-column: span 2 }
.field.compact{ width: 220px }
.field label{ font-size:.875rem; color: var(--muted); font-weight: 600; letter-spacing:.2px }
.field input, .field select, .field textarea{
  border: 1.5px solid var(--border); border-radius: var(--radius-s);
  padding: 10px 12px; outline: none; background: var(--panel-2); color: var(--text);
}
.field textarea{ resize: vertical }
.field input::placeholder, .field textarea::placeholder { color:#9aa3b2 }
.field input:focus, .field select:focus, .field textarea:focus{
  border-color: var(--accent); box-shadow: 0 0 0 3px #dbeafe;
}
.row{ display:flex; gap:8px }

.segmented{
  background: #eef2f7; border: 1px solid var(--border); border-radius: var(--radius-s);
  padding: 4px; display:flex; gap:4px
}
.segmented button{
  border:0; background: transparent; border-radius: 10px; padding: 6px 10px; cursor:pointer; color: var(--muted)
}
.segmented button.on{
  background:#fff; color: var(--text); border:1px solid var(--border); box-shadow: var(--shadow-s)
}

.counter{ justify-self:end; align-self:center }
.chip{
  display:inline-block; padding: 6px 10px; border-radius: 999px;
  background:#eaf2ff; color:#1e40af; border: 1px solid #d5e4ff; font-weight:700; font-size:.85rem
}

/* Cards */
.cards{
  padding: 16px 18px; display:grid; grid-template-columns: repeat(auto-fill,minmax(320px,1fr)); gap: 16px;
}
.empty{
  grid-column: 1 / -1; text-align:center; color: var(--muted); padding: 28px;
  border: 2px dashed var(--border); border-radius: var(--radius-m); background: #f7fafc;
}
.card{
  background: var(--panel); border: 1px solid var(--border); border-radius: var(--radius-l);
  padding: 16px; box-shadow: var(--shadow-m); transition:.15s;
}
.card:hover{ transform: translateY(-2px) }
.card-head{ display:flex; justify-content:space-between; align-items:center; gap:8px; margin-bottom: 10px }
.person{ display:flex; gap:10px; align-items:center }
.avatar{ width: 44px; height: 44px; display:grid; place-items:center; border-radius: 12px; background: #eef2ff; color:#1e40af; font-weight:800 }
.name{ font-weight:800; letter-spacing:.2px; font-size: 1.05rem }
.id{ font-size:.875rem; color: var(--muted) }
.grid{ display:grid; grid-template-columns: 1fr 1fr; gap:8px; margin-top:6px }
.activities{ margin-top: 8px; display:flex; gap: 6px; flex-wrap: wrap }
.act-chip{ background:#f0f7ff; color:#284b8a; border:1px solid #e0e9ff; padding: 4px 8px; border-radius: 999px; font-size:.8rem }
.act-note{ color: var(--muted); font-size: .95rem }
.card-actions{ display:flex; justify-content:flex-end; gap:10px; margin-top: 12px }

/* Tipo (pill) */
.pill{
  display:inline-block; padding: 4px 10px; font-size:.8rem; font-weight:800; border-radius:999px; border:1px solid var(--border);
  letter-spacing:.2px;
}
.pill--planta{ background:#ecfdf5; color:#166534; border-color:#a7f3d0 }
.pill--catedra{ background:#fffbeb; color:#92400e; border-color:#fde68a }
.pill--contrato{ background:#eff6ff; color:#1e40af; border-color:#bfdbfe }
.pill--neutral{ background:#f1f5f9; color:#334155 }

/* Tabla */
.table-card{ padding: 12px 18px }
.table-wrapper{
  overflow:auto; border-radius: var(--radius-m); border: 1px solid var(--border);
  background: var(--panel); box-shadow: var(--shadow-m)
}
.table{ width:100%; border-collapse: collapse; font-size: 0.975rem }
.table th, .table td{ padding: 12px; border-bottom: 1px solid var(--border) }
.table thead th{
  text-align:left; color: var(--muted); background:#f3f6fa; position: sticky; top: 0; z-index: 1; border-bottom:1px solid var(--border-strong)
}
.table tbody tr:nth-child(odd){ background: #fcfdff }
.table tbody tr:hover{ background: #eef4ff }
.table .empty{ text-align:center; color: var(--muted) }
.row-actions{ display:flex; justify-content:flex-end; gap:8px }

/* Planner (vista semanal) */
.planner{ padding: 12px 18px; display:grid; gap:12px }
.planner-head{
  display:flex; align-items:center; justify-content:space-between;
  background: var(--panel); border:1px solid var(--border); border-radius: var(--radius-m); padding: 10px 12px; box-shadow: var(--shadow-s)
}
.legend{ display:flex; gap:8px; flex-wrap:wrap }
.legend-item{ padding: 4px 10px; border-radius: 999px; font-size:.8rem; border:1px solid var(--border); font-weight:700 }
.legend-item.planta{ background:#ecfdf5; color:#166534; border-color:#a7f3d0 }
.legend-item.catedra{ background:#fffbeb; color:#92400e; border-color:#fde68a }
.legend-item.contrato{ background:#eff6ff; color:#1e40af; border-color:#bfdbfe }

.planner-grid{ display:grid; grid-template-columns: 84px 1fr; gap:12px }
.hours-rail{
  background: var(--panel); border:1px solid var(--border); border-radius: var(--radius-m); padding: 8px; box-shadow: var(--shadow-s)
}
.hour{ height: 60px; display:flex; align-items:flex-start; justify-content:flex-end; padding-right:6px; color: var(--muted); font-size:.85rem }

.days{
  position:relative; background: var(--panel); border:1px solid var(--border); border-radius: var(--radius-m);
  overflow:auto; padding-top: 34px; box-shadow: var(--shadow-s)
}
.day-header{
  position: sticky; top: 0; height: 34px; display:inline-flex; align-items:center; justify-content:center;
  color:#263241; font-weight:700; font-size:.85rem; border-right:1px solid var(--border);
  width: calc(100%/7); background:#f7f9fc; float:left;
}
.day-col{
  position:relative; width: calc(100%/7); float:left; border-right:1px dashed #e8edf3;
  background-image: repeating-linear-gradient(to bottom, #eef2f7 0 1px, transparent 1px 60px);
}
.hour-line{ display:none }

.block{
  position:absolute; left:6px; right:6px; border-radius: 10px; padding: 6px 8px;
  color:#1f2937; font-size:.85rem; overflow:hidden; border: 1px solid #dfe6ef; box-shadow: var(--shadow-s);
}
.block .b-title{ font-weight:800 }
.block .b-sub{ color: var(--muted); font-weight:600 }
.k-planta{ background:#d9fbe7 }
.k-catedra{ background:#fff0c8 }
.k-contrato{ background:#deebff }

/* Toasts */
.toasts{ position: fixed; right: 16px; bottom: 16px; display:grid; gap:8px; z-index:60 }
.toast{
  background:#ffffff; color:#263241; border:1.5px solid var(--border); padding: 10px 12px;
  border-radius: var(--radius-s); box-shadow: var(--shadow-l)
}

/* ===== MODALES (corregido: SIN transparencia) ===== */
.overlay{
  position: fixed; inset: 0; display:grid; place-items:center;
  background: rgba(0,0,0,.55);   /* oscurece el fondo de forma clara */
  padding:16px; z-index:1000;    /* por encima de todo */
}
.modal{
  width: min(980px, 96vw);
  background: #ffffff;          /* panel SÓLIDO */
  border: 1px solid var(--border-strong);
  border-radius: var(--radius-l);
  padding: 16px;
  box-shadow: var(--shadow-l);
  max-height: 92vh; overflow: auto;
}

/* Modal contenido */
.modal-head{ display:flex; align-items:start; justify-content:space-between; gap:12px; margin-bottom:12px }
.form-grid{ display:grid; grid-template-columns: 1fr 1fr; gap: 12px }
.form-grid .col-2{ grid-column: 1 / -1 }
.subhead{ display:flex; align-items:center; justify-content:space-between; gap:8px; margin-top: 2px }
.sub-actions{ display:flex; gap:6px; flex-wrap:wrap }
.horario{ display:grid; grid-template-columns: 140px 120px 120px 1fr 1fr; gap:8px; margin-top: 8px }
.horario-right{ display:flex; gap:8px }
.actividad-row{ display:grid; grid-template-columns: 2fr .7fr 2fr auto; gap:8px; margin-top:8px }
.form-actions{ display:flex; justify-content:flex-end; gap:8px; padding-top:6px }
.pre{ white-space: pre-wrap }

.detail .detail-top{ display:flex; justify-content:space-between; align-items:center }
.detail .detail-name{ margin: 6px 0 10px; font-size: 1.1rem; font-weight: 900 }
.detail .detail-grid{ display:grid; grid-template-columns: 1fr 1fr; gap: 12px }
.print-card{ break-inside: avoid; page-break-inside: avoid; border: 1px solid var(--border); border-radius: var(--radius-m); padding: 12px }

/* ===== Sidebar (si tu plantilla usa .app/.sidebar) - todo sólido ===== */
.app{ min-height:100vh; display:grid; grid-template-columns: 260px 1fr; color: var(--text); background: var(--bg) }
.sidebar{
  position: sticky; top:0; height:100vh; padding:18px; border-right:1px solid var(--border);
  background: #ffffff; display:flex; flex-direction:column; gap:16px
}
.brand, .panel, .menu-item{ background:#fff; border:1px solid var(--border); border-radius: var(--radius-m); box-shadow: var(--shadow-s) }
.menu{ display:grid; gap:8px }
.menu-item{ display:flex; align-items:center; gap:10px; padding: 10px 12px; color: var(--text); cursor:pointer; transition:.15s }
.menu-item:hover{ background:#f7faff }
.menu-item .icon{ width:18px; text-align:center }
.menu-item.danger{ color:#fff; background: var(--danger); border-color: var(--danger) }
.menu-item.file{ position:relative; overflow:hidden }
.menu-item.file input[type="file"]{ position:absolute; inset:0; opacity:0; cursor:pointer }

/* Responsive */
@media (max-width: 980px){
  .filters{ grid-template-columns: 1fr }
  .grid{ grid-template-columns: 1fr }
  .form-grid{ grid-template-columns: 1fr }
  .horario{ grid-template-columns: 1fr 1fr }
  .actividad-row{ grid-template-columns: 1fr 1fr }
  .days, .hours-rail{ max-height: 60vh }
  .app{ grid-template-columns: 1fr }
  .sidebar{ position: static; height: auto }
}

/* Print */
@media print{
  .shell{ background:#fff; color:#000 }
  .table thead th{ background:#fff }
  .no-print{ display:none !important }
}
</style>
