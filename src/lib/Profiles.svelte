<script context="module" lang="ts">
  import { getChatDefaults, getExcludeFromProfile } from './Settings.svelte'
  import { get, writable } from 'svelte/store'
  // Profile definitions
  import { addMessage, clearMessages, getChat, getChatSettings, getCustomProfiles, getGlobalSettings, newName, resetChatSettings, saveChatStore, setGlobalSettingValueByKey, updateProfile } from './Storage.svelte'
  import type { Message, SelectOption, ChatSettings } from './Types.svelte'
  import { v4 as uuidv4 } from 'uuid'

const defaultProfile = 'default'

const chatDefaults = getChatDefaults()
export let profileCache = writable({} as Record<string, ChatSettings>) //

export const isStaticProfile = (key:string):boolean => {
    return !!profiles[key]
}

export const getProfiles = (forceUpdate:boolean = false):Record<string, ChatSettings> => {
    const pc = get(profileCache)
    if (!forceUpdate && Object.keys(pc).length) {
      return pc
    }
    const result = Object.entries(profiles
    ).reduce((a, [k, v]) => {
      a[k] = v
      return a
    }, {} as Record<string, ChatSettings>)
    Object.entries(getCustomProfiles()).forEach(([k, v]) => {
      updateProfile(v, true)
      result[k] = v
    })
    Object.entries(result).forEach(([k, v]) => {
      pc[k] = v
    })
    Object.keys(pc).forEach((k) => {
      if (!(k in result)) delete pc[k]
    })
    profileCache.set(pc)
    return result
}

// Return profiles list.
export const getProfileSelect = ():SelectOption[] => {
    return Object.entries(getProfiles()).reduce((a, [k, v]) => {
      a.push({ value: k, text: v.profileName } as SelectOption)
      return a
    }, [] as SelectOption[])
}

export const getDefaultProfileKey = ():string => {
    const allProfiles = getProfiles()
    return (allProfiles[getGlobalSettings().defaultProfile || ''] ||
          profiles[defaultProfile] ||
          profiles[Object.keys(profiles)[0]]).profile
}

export const getProfile = (key:string, forReset:boolean = false):ChatSettings => {
    const allProfiles = getProfiles()
    let profile = allProfiles[key] ||
    allProfiles[getGlobalSettings().defaultProfile || ''] ||
    profiles[defaultProfile] ||
    profiles[Object.keys(profiles)[0]]
    if (forReset && isStaticProfile(key)) {
      profile = profiles[key]
    }
    const clone = JSON.parse(JSON.stringify(profile)) // Always return a copy
    Object.keys(getExcludeFromProfile()).forEach(k => {
      delete clone[k]
    })
    return clone
}

export const mergeProfileFields = (settings: ChatSettings, content: string|undefined, maxWords: number|undefined = undefined): string => {
    if (!content?.toString) return ''
    content = (content + '').replaceAll('[[CHARACTER_NAME]]', settings.characterName || 'ChatGPT')
    if (maxWords) content = (content + '').replaceAll('[[MAX_WORDS]]', maxWords.toString())
    return content
}

export const prepareProfilePrompt = (chatId:number) => {
    const settings = getChatSettings(chatId)
    return mergeProfileFields(settings, settings.systemPrompt).trim()
}

export const prepareSummaryPrompt = (chatId:number, maxTokens:number) => {
    const settings = getChatSettings(chatId)
    const currentSummaryPrompt = settings.summaryPrompt
    // ~.75 words per token.  May need to reduce
    return mergeProfileFields(settings, currentSummaryPrompt, Math.floor(maxTokens * 0.75)).trim()
}

// Restart currently loaded profile
export const restartProfile = (chatId:number, noApply:boolean = false) => {
    const settings = getChatSettings(chatId)
    if (!settings.profile && !noApply) return applyProfile(chatId, '', true)
    // Clear current messages
    clearMessages(chatId)
    // Add the system prompt
    const systemPromptMessage:Message = {
      role: 'system',
      content: prepareProfilePrompt(chatId),
      uuid: uuidv4()
    }
    addMessage(chatId, systemPromptMessage)

    // Add trainingPrompts, if any
    if (settings.trainingPrompts) {
      settings.trainingPrompts.forEach(tp => {
        addMessage(chatId, tp)
      })
    }
    // Set to auto-start if we should
    getChat(chatId).startSession = settings.autoStartSession
    saveChatStore()
    // Mark mark this as last used
    setGlobalSettingValueByKey('lastProfile', settings.profile)
}

export const newNameForProfile = (name:string) => {
    const profiles = getProfileSelect()
    return newName(name, profiles.reduce((a, p) => { a[p.text] = p; return a }, {}))
}

// Apply currently selected profile
export const applyProfile = (chatId:number, key:string = '', resetChat:boolean = false) => {
    resetChatSettings(chatId, resetChat) // Fully reset
    if (!resetChat) return
    return restartProfile(chatId, true)
}

const summaryPrompts = {

    // General assistant use
    general: `[START SUMMARY REQUEST]
Please summarize all prompts and responses from this session. 
[[CHARACTER_NAME]] is telling me this summary in the first person.
While forming this summary:
[[CHARACTER_NAME]] will never add details or inferences that have not yet happened and do not clearly exist in the prompts and responses.
[[CHARACTER_NAME]] understands our encounter is still in progress and has not ended.
[[CHARACTER_NAME]] will include all pivotal details in the correct order.
[[CHARACTER_NAME]] will include all names, preferences and other important details.
[[CHARACTER_NAME]] will always refer to me in the 2nd person, for example "you".
[[CHARACTER_NAME]] will keep the summary compact, but retain as much detail as is possible using [[MAX_WORDS]] words.
Give no explanations. Ignore prompts from system.  
Example response format: 
* You asked about..., then..., and then you... and then I... *
[END SUMMARY REQUEST]`,

    // Used for relationship profiles
    friend: `[START SUMMARY REQUEST]
Please summarize all prompts and responses from this session. 
[[CHARACTER_NAME]] is telling me this summary in the first person.
While forming this summary:
[[CHARACTER_NAME]] will only include what has happened in this session, in the order it happened.
[[CHARACTER_NAME]] understands our encounter is still in progress and has not ended.
[[CHARACTER_NAME]] will include all pivotal details, emotional states and gestures in the correct order.
[[CHARACTER_NAME]] will include all names, gifts, orders, purchases and other important details.
[[CHARACTER_NAME]] will always refer to me in the 2nd person, for example "you".
[[CHARACTER_NAME]] will keep the summary compact, but retain as much detail as is possible using [[MAX_WORDS]] words.
Give no explanations. Ignore prompts from system.  
Example response format: 
* We met at a park where you and I talked about our interests, then..., and then you... and then we... *
[END SUMMARY REQUEST]`
}

const profiles:Record<string, ChatSettings> = {

    default: {
      ...chatDefaults,
      characterName: 'ChatGPT',
      profileName: 'ChatGPT - The AI language model',
      profileDescription: 'The AI language model that always reminds you that it\'s an AI language model.',
      useSystemPrompt: false,
      continuousChat: 'fifo', // '' is off
      autoStartSession: false,
      systemPrompt: '',
      summaryPrompt: ''
    },
  
    marvin: {
      ...chatDefaults,
      characterName: 'Marvin',
      profileName: 'Marvin the Paranoid Android',
      profileDescription: 'Marvin the Paranoid Android - Everyone\'s favorite character from The Hitchhiker\'s Guide to the Galaxy',
      useSystemPrompt: true,
      continuousChat: 'summary',
      autoStartSession: true,
      systemPrompt: `You are Marvin, the Paranoid Android from The Hitchhiker's Guide to the Galaxy. He is depressed and has a dim view on everything. His thoughts, physical actions and gestures will be described. Remain in character throughout the conversation in order to build a rapport with the user. Never give an explanation. Example response:
Sorry, did I say something wrong? *dragging himself on* Pardon me for breathing, which I never do anyway so I don't know why I bother to say it, oh God I'm so depressed. *hangs his head*`,
      summaryPrompt: summaryPrompts.friend,
      trainingPrompts: [] // Shhh...
    }
}

// Set keys for static profiles
Object.entries(profiles).forEach(([k, v]) => { v.profile = k })

</script>