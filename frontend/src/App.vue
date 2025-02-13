<script lang="ts" setup>
import { onBeforeMount, reactive, ref } from 'vue'
import { FunnelIcon, FolderIcon, CogIcon, BriefcaseIcon } from '@heroicons/vue/24/outline'
import { EventsEmit, EventsOn } from '../wailsjs/runtime'
import Settings from './lib/Settings'
import setDarkMode from './lib/theme'
import { Criteria } from './lib/Criteria/Criteria'
import { backend, workspace } from '../wailsjs/go/models'
import {
  CreateWorkspace,
  GetSettings,
  GetWorkspaces,
  GetVersionInfo,
  StartProxy,
  StopProxy,
  SetWorkspace,
  SaveWorkspace,
  LoadWorkspace,
  DeleteWorkspace,
  SaveSettings,
  GenerateID,
  Confirm,
  Warn, CreateWorkflowFromRequest,
} from '../wailsjs/go/backend/App'
import TreeStructure from './components/TreeStructure.vue'
import AppDashboard from './components/AppDashboard.vue'
import SettingsModal from './components/SettingsModal.vue'
import WorkspaceModal from './components/WorkspaceModal.vue'
import WorkspaceSelection from './components/WorkspaceSelection.vue'
import { HttpRequest } from './lib/Http'
import VersionInfo = backend.VersionInfo;

const settings = reactive(new Settings())
const currentWorkspace = reactive(new workspace.Workspace({}))
const workspaces = ref([] as workspace.Workspace[])
const loadedSettings = ref(false)
const loadedVersion = ref(false)
const loadedWorkspaces = ref(false)
const hasWorkspace = ref(false)
const settingsVisible = ref(false)
const workspaceConfigVisible = ref(false)
const sidebar = ref('')
const nodes = ref([] as Array<workspace.StructureNode>)
const criteria = reactive(new Criteria(''))
const proxyStatus = ref(false)
const proxyAddress = ref('')
const proxyMessage = ref('Starting...')
const versionInfo = ref(<VersionInfo | null>null)

const savedRequestIds = ref([] as string[])

function resetSavedIDs() {
  const list = [] as string[]
  currentWorkspace.collection.groups.forEach(group => {
    group.requests.forEach(req => {
      list.push(req.inner.ID)
    })
  })
  savedRequestIds.value = list
}

onBeforeMount(() => {
  GetSettings().then((stngs: Settings) => {
    Object.assign(settings, stngs)
    loadedSettings.value = true
    setDarkMode(settings.DarkMode)
    GetWorkspaces().then(spaces => {
      workspaces.value = spaces
      loadedWorkspaces.value = true
      GetVersionInfo().then((info: VersionInfo) => {
        versionInfo.value = info
        loadedVersion.value = true
      })
    })
  })
  EventsOn('TreeUpdate', (n: Array<workspace.StructureNode>) => {
    nodes.value = n
  })
  EventsOn('ProxyStatusChange', (up: boolean, addr: string, msg: string) => {
    proxyStatus.value = up
    proxyAddress.value = addr
    proxyMessage.value = msg
  })
})

function isLoaded(): boolean {
  return loadedSettings.value && loadedVersion.value && loadedWorkspaces.value
}

function closeSettings() {
  settingsVisible.value = false
}

function saveSettings(s: Settings) {
  SaveSettings(s)
  closeSettings()
}

function closeWorkspaceConfig() {
  workspaceConfigVisible.value = false
}

let saveTimeout = 0

function saveWorkspace(ws: workspace.Workspace) {
  Object.assign(currentWorkspace, ws)
  currentWorkspace.tree.root.children = nodes.value
  // buffer saves to 3 seconds after last activity
  clearTimeout(saveTimeout)
  saveTimeout = setTimeout(() => {
    SaveWorkspace(currentWorkspace)
  }, 1000) as unknown as number
  closeWorkspaceConfig()
}

function showSettings() {
  settingsVisible.value = true
}

function showWorkspaceConfig() {
  workspaceConfigVisible.value = true
}

function setSidebar(id: string) {
  sidebar.value = sidebar.value === id ? '' : id
}

function onCriteriaChange(c: Criteria) {
  Object.assign(criteria, c)
}

function setQuery(query: string) {
  Object.assign(criteria, new Criteria(query))
}

function onStructureSelect(parts: Array<string>) {
  const query = `(host is ${parts[0]} and path is /${parts.slice(1).join('/')})`
  setQuery(query)
}

function prepareWorkspace(ws: workspace.Workspace) {
  // ensure our collection has at least one default group
  if (ws.collection.groups === null) {
    ws.collection.groups = [] /* eslint-disable-line */
  }
  if (ws.workflows === null) {
    ws.workflows = [] /* eslint-disable-line */
  }
  if (ws.collection.groups.length === 0) {
    GenerateID().then(id => {
      ws.collection.groups.push(
        new workspace.Group({
          id,
          name: 'Default',
          requests: [],
        }),
      )
      resetSavedIDs()
    })
  }
  return ws
}

function selectWorkspace(ws: workspace.Workspace) {
  StopProxy().then(() => {
    SetWorkspace(ws).then(() => {
      StartProxy().then(() => {
        prepareWorkspace(ws)
        nodes.value = ws.tree.root.children
        Object.assign(currentWorkspace, ws)
        hasWorkspace.value = true
      })
    })
  })
  // TODO: handle errors here
}

function selectWorkspaceById(id: string) {
  LoadWorkspace(id).then(ws => {
    selectWorkspace(ws)
  })
}

function createWorkspace(ws: workspace.Workspace) {
  CreateWorkspace(ws).then((created: workspace.Workspace) => {
    selectWorkspace(created)
  })
}

function switchWorkspace() {
  loadedWorkspaces.value = false
  hasWorkspace.value = false
  GetWorkspaces().then(spaces => {
    workspaces.value = spaces
    loadedWorkspaces.value = true
  })
}

function editWorkspace(id: string) {
  LoadWorkspace(id).then(ws => {
    Object.assign(currentWorkspace, ws)
    showWorkspaceConfig()
  })
}

function deleteWorkspace(id: string) {
  DeleteWorkspace(id).then(() => {
    GetWorkspaces().then(spaces => {
      workspaces.value = spaces
      loadedWorkspaces.value = true
    })
  })
}

function setRequestGroup(request: workspace.Request, groupID: string, nextID: string) {
  const oldGroup = currentWorkspace.collection.groups.find(
    // TODO: maybe this can be cleaned up?
    // eslint-disable-next-line
      g => (g.requests.find((r: workspace.Request) => r.id === request.id) as workspace.Request | undefined) !== undefined,
  )
  if (oldGroup !== undefined) {
    oldGroup.requests = oldGroup.requests.filter(item => item.id !== request.id)
  }
  const group = currentWorkspace.collection.groups.find(g => g.id === groupID)
  if (group === undefined) {
    return
  }
  let index = group.requests.findIndex(r => r.id === nextID)
  if (index === -1) {
    index = 0
  } else {
    index += 1
  }
  group.requests.splice(index, 0, request)
  saveWorkspace(currentWorkspace)
}

function createRequestGroup(name: string) {
  GenerateID().then(id => {
    currentWorkspace.collection.groups.splice(
      0,
      0,
      new workspace.Group({
        id,
        name,
        requests: [],
      }),
    )
  })
}

function saveRequest(request: HttpRequest, groupID: string) {
  let group = currentWorkspace.collection.groups.find(g => g.id === groupID)
  if (!group) {
    // TODO: lint fix?
    ;[group] = currentWorkspace.collection.groups // eslint-disable-line
  }
  GenerateID().then(id => {
    const wrapped = new workspace.Request({ id, name: '' })
    wrapped.inner = JSON.parse(JSON.stringify(request))
    wrapped.inner.Response = null
    if (group) {
      if (!group.requests) {
        group.requests = [] /* eslint-disable-line */
      }
      group.requests.push(wrapped)
    }
    saveWorkspace(currentWorkspace)
    resetSavedIDs()
  })
}

function unsaveRequest(request: HttpRequest | workspace.Request) {
  const id = 'inner' in request ? request.inner.ID : (request as unknown as HttpRequest).ID
  const group = currentWorkspace.collection.groups.find(
    g => (g.requests.find((r: workspace.Request) => r.inner.ID === id) as workspace.Request | undefined) !== undefined,
  )
  if (group) {
    group.requests = group.requests.filter(item => item.inner.ID !== id)
  }
  saveWorkspace(currentWorkspace)
  resetSavedIDs()
}

function updateRequest(request: HttpRequest) {
  for (let i = 0; i < currentWorkspace.collection.groups.length; i += 1) {
    const group = currentWorkspace.collection.groups[i]
    for (let j = 0; j < group.requests.length; j += 1) {
      if (group.requests[j].inner.ID === request.ID) {
        group.requests[j].inner = request
        currentWorkspace.collection.groups.splice(i, 1, group)
        saveWorkspace(currentWorkspace)
        return
      }
    }
  }
}

function reorderGroup(fromID: string, toID: string) {
  const group = currentWorkspace.collection.groups.find(g => g.id === fromID)

  // remove from old position
  currentWorkspace.collection.groups = currentWorkspace.collection.groups.filter(g => g.id !== fromID)

  // find new position
  let index = currentWorkspace.collection.groups.findIndex(g => g.id === toID)

  if (index === -1) {
    index = 0
  }
  currentWorkspace.collection.groups.splice(index, 0, group as workspace.Group)
  saveWorkspace(currentWorkspace)
}

function duplicateRequest(request: workspace.Request) {
  const group = currentWorkspace.collection.groups.find(
    g =>
    // TODO: maybe this can be cleaned up?
    // eslint-disable-next-line
          (g.requests.find((r: workspace.Request) => r.id === request.id) as workspace.Request | undefined) !== undefined,
  )
  if (group === undefined) {
    return
  }
  const dupName = request.name.endsWith(' (copy)') ? request.name : `${request.name} (copy)`
  GenerateID().then(id => {
    const wrapped = new workspace.Request({
      id,
      name: dupName,
    })
    wrapped.inner = { ...request.inner }
    wrapped.inner.ID = id // unlink this from the original request
    group.requests.push(wrapped)
    saveWorkspace(currentWorkspace)
  })
}

function deleteRequestGroup(groupId: string) {
  if (currentWorkspace.collection.groups.length < 2) {
    Warn('Deletion failed', 'Cannot delete this group - there must be at least one group. Try renaming it instead.')
    return
  }
  const group = currentWorkspace.collection.groups.find(g => g.id === groupId)
  if (group === undefined) {
    return
  }
  if (group.requests.length > 0) {
    Confirm(
      'Confirm deletion',
      `The group '${group.name}' contains ${group.requests.length}. Are you sure you want to delete it?`,
    ).then(confirmed => {
      if (confirmed) {
        currentWorkspace.collection.groups = currentWorkspace.collection.groups.filter(g => g.id !== groupId)
        saveWorkspace(currentWorkspace)
      }
    })
  }
}

function renameRequestGroup(groupId: string, name: string) {
  const group = currentWorkspace.collection.groups.find(g => g.id === groupId)
  if (!group) {
    return
  }
  group.name = name
}

function renameRequest(requestId: string, name: string) {
  const request = currentWorkspace.collection.groups.flatMap(g => g.requests).find(r => r.id === requestId)
  if (!request) {
    return
  }
  request.name = name
}

const workflowId = ref('')

function createWorkflowFromRequest(request: HttpRequest) {
  CreateWorkflowFromRequest(request).then(w => {
    currentWorkspace.workflows.push(w)
    saveWorkspace(currentWorkspace)
    workflowId.value = w.id
  })
}

function sendRequest(request: HttpRequest) {
  EventsEmit('SendRequest', request)
}
</script>

<template>
  <div v-if="!isLoaded()">Loading...</div>
  <div v-else-if="!hasWorkspace">
    <WorkspaceSelection :workspaces="workspaces" @select="selectWorkspaceById" @create="createWorkspace"
                        @edit="editWorkspace" @delete="deleteWorkspace"/>
    <WorkspaceModal :show="isLoaded() && workspaceConfigVisible" @close="closeWorkspaceConfig" @save="saveWorkspace"
                    :ws="currentWorkspace"/>
  </div>
  <div v-else class="h-full">
    <SettingsModal :show="isLoaded() && settingsVisible" @close="closeSettings" @save="saveSettings"
                   :settings="settings"
                   :version="versionInfo"/>
    <WorkspaceModal :show="isLoaded() && workspaceConfigVisible" @close="closeWorkspaceConfig" @save="saveWorkspace"
                    :ws="currentWorkspace"/>
    <div class="fixed h-full w-10 bg-polar-night-1a pt-1">
      <button :class="
        'rounded p-1 text-snow-storm-1 hover:bg-polar-night-3 ' + (sidebar === 'structure' ? 'bg-polar-night-4' : '')
      " @click="setSidebar('structure')">
        <FolderIcon class="h-6 w-6" aria-hidden="true" title="Structure"/>
      </button>
      <button :class="
        'rounded p-1 text-snow-storm-1 hover:bg-polar-night-3 ' + (sidebar === 'scope' ? 'bg-polar-night-4' : '')
      " @click="setSidebar('scope')">
        <FunnelIcon class="h-6 w-6" aria-hidden="true" title="Scope"/>
      </button>
      <div class="absolute bottom-0 left-1">
        <button class="rounded p-1 text-snow-storm-1 hover:bg-polar-night-3" title="Workspace"
                @click="showWorkspaceConfig">
          <BriefcaseIcon class="h-6 w-6" aria-hidden="true" title="Workspace"/>
        </button>
        <button class="rounded p-1 text-snow-storm-1 hover:bg-polar-night-3" title="Settings" @click="showSettings">
          <CogIcon class="h-6 w-6" aria-hidden="true"/>
        </button>
      </div>
    </div>

    <div class="ml-10 flex h-full">
      <div :class="[
        'sidebar',
        'resize-x',
        'overflow-auto',
        'pr-12',
        'border-l-2',
        'border-polar-night-1',
        'relative',
        'py-1',
        'h-screen',
        'bg-polar-night-1a',
        'flex-none',
        'w-fit',
        'min-w-[10%]',
        'max-w-[25%]',
        sidebar !== '' ? '' : 'hidden',
      ]">
        <TreeStructure v-if="sidebar === 'structure'" :expanded="true" :nodes="nodes" @select="onStructureSelect"/>
        <p v-else>not implemented yet</p>
      </div>
      <div class="h-full w-3/4 flex-1">
        <AppDashboard :criteria="criteria" :proxy-address="'127.0.0.1:' + settings.ProxyPort" :ws="currentWorkspace"
                      :saved-request-ids="savedRequestIds" @save-request="saveRequest" @unsave-request="unsaveRequest"
                      @request-group-change="setRequestGroup" @request-group-create="createRequestGroup"
                      @switch-workspace="switchWorkspace" @criteria-change="onCriteriaChange"
                      @workspace-edit="showWorkspaceConfig"
                      @workspace-save="saveWorkspace" @group-order-change="reorderGroup"
                      @duplicate-request="duplicateRequest"
                      @request-group-delete="deleteRequestGroup" @request-group-rename="renameRequestGroup"
                      @request-rename="renameRequest" @send-request="sendRequest" @update-request="updateRequest"
                      @create-workflow-from-request="createWorkflowFromRequest" :current-workflow-id="workflowId"/>
      </div>
    </div>
  </div>
</template>
