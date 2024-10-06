//GET USER INPUT/PROMPT AND FORMAT IT --> script.js
function extractMessageFromData(data) {
    if (typeof data === 'string') {
        return data;
    }

    switch (main_api) {
        case 'kobold':
            return data.results[0].text;

        //Two main sources
        case 'koboldhorde':
            return data.text;
        case 'textgenerationwebui':
            return data.choices?.[0]?.text ?? data.content ?? data.response ?? '';

        case 'novel':
            return data.output;
        case 'openai':
            return data?.choices?.[0]?.message?.content ?? data?.choices?.[0]?.text ?? data?.text ?? '';
        default:
            return '';
    }
}

//WHERE THE CHARACTER PERSONA IS ADDED --> script.js
export function substituteParams(content, _name1, _name2, _original, _group, _replaceCharacterCard = true, additionalMacro = {}) {
    if (!content) {
        return '';
    }

    const environment = {};

    if (typeof _original === 'string') {
        let originalSubstituted = false;
        environment.original = () => {
            if (originalSubstituted) {
                return '';
            }

            originalSubstituted = true;
            return _original;
        };
    }

    const getGroupValue = () => {
        if (typeof _group === 'string') {
            return _group;
        }

        if (selected_group) {
            const members = groups.find(x => x.id === selected_group)?.members;
            const names = Array.isArray(members)
                ? members.map(m => characters.find(c => c.avatar === m)?.name).filter(Boolean).join(', ')
                : '';
            return names;
        } else {
            return _name2 ?? name2;
        }
    };

    if (_replaceCharacterCard) {
        const fields = getCharacterCardFields();
        environment.charPrompt = fields.system || '';
        environment.charJailbreak = fields.jailbreak || '';
        environment.description = fields.description || '';
        environment.personality = fields.personality || '';
        environment.scenario = fields.scenario || '';
        environment.persona = fields.persona || '';
        environment.mesExamples = fields.mesExamples || '';
        environment.charVersion = fields.version || '';
        environment.char_version = fields.version || '';
    }

    // Must be substituted last so that they're replaced inside {{description}}
    environment.user = _name1 ?? name1;
    environment.char = _name2 ?? name2;
    environment.group = environment.charIfNotGroup = getGroupValue();
    environment.model = getGeneratingModel();

    if (additionalMacro && typeof additionalMacro === 'object') {
        Object.assign(environment, additionalMacro);
    }

    return evaluateMacros(content, environment);
}

//AI TEXT GENERATION --> script.js
export async function generateRaw(prompt, api, instructOverride, quietToLoud, systemPrompt, responseLength) {
    if (!api) {
        api = main_api;
    }

    const abortController = new AbortController();
    const responseLengthCustomized = typeof responseLength === 'number' && responseLength > 0;
    let originalResponseLength = -1;
    const isInstruct = power_user.instruct.enabled && api !== 'openai' && api !== 'novel' && !instructOverride;
    const isQuiet = true;

    if (systemPrompt) {
        systemPrompt = substituteParams(systemPrompt);
        systemPrompt = isInstruct ? formatInstructModeSystemPrompt(systemPrompt) : systemPrompt;
        prompt = api === 'openai' ? prompt : `${systemPrompt}\n${prompt}`;
    }

    prompt = substituteParams(prompt);
    prompt = api == 'novel' ? adjustNovelInstructionPrompt(prompt) : prompt;
    prompt = isInstruct ? formatInstructModeChat(name1, prompt, false, true, '', name1, name2, false) : prompt;
    prompt = isInstruct ? (prompt + formatInstructModePrompt(name2, false, '', name1, name2, isQuiet, quietToLoud)) : (prompt + '\n');

    try {
        originalResponseLength = responseLengthCustomized ? saveResponseLength(api, responseLength) : -1;
        let generateData = {};

        switch (api) {
            case 'kobold':
            case 'koboldhorde':
                if (preset_settings === 'gui') {
                    generateData = { prompt: prompt, gui_settings: true, max_length: amount_gen, max_context_length: max_context, api_server };
                } else {
                    const isHorde = api === 'koboldhorde';
                    const koboldSettings = koboldai_settings[koboldai_setting_names[preset_settings]];
                    generateData = getKoboldGenerationData(prompt, koboldSettings, amount_gen, max_context, isHorde, 'quiet');
                }
                break;
            case 'novel': {
                const novelSettings = novelai_settings[novelai_setting_names[nai_settings.preset_settings_novel]];
                generateData = getNovelGenerationData(prompt, novelSettings, amount_gen, false, false, null, 'quiet');
                break;
            }
            case 'textgenerationwebui':
                generateData = getTextGenGenerationData(prompt, amount_gen, false, false, null, 'quiet');
                break;
            case 'openai': {
                generateData = [{ role: 'user', content: prompt.trim() }];
                if (systemPrompt) {
                    generateData.unshift({ role: 'system', content: systemPrompt.trim() });
                }
            } break;
        }

        let data = {};

        //Where the input is passed into AI to generate response
        //KoboldHorde
        if (api == 'koboldhorde') {
            data = await generateHorde(prompt, generateData, abortController.signal, false);
        } 
    
        //OpenAI    
        else if (api == 'openai') {
            data = await sendOpenAIRequest('quiet', generateData, abortController.signal);
        } 
        
        //Everything Else
        else {
            const generateUrl = getGenerateUrl(api);
            const response = await fetch(generateUrl, {
                method: 'POST',
                headers: getRequestHeaders(),
                cache: 'no-cache',
                body: JSON.stringify(generateData),
                signal: abortController.signal,
            });

            if (!response.ok) {
                const error = await response.json();
                throw error;
            }

            data = await response.json();
        }

        if (data.error) {
            throw new Error(data.response);
        }

        const message = cleanUpMessage(extractMessageFromData(data), false, false, true);

        if (!message) {
            throw new Error('No message generated');
        }

        return message;
    } finally {
        if (responseLengthCustomized) {
            restoreResponseLength(api, originalResponseLength);
        }
    }
}

//GET SETTINGS AT START --> script.js
export async function getSettings() {
    const response = await fetch('/api/settings/get', {
        method: 'POST',
        headers: getRequestHeaders(),
        body: JSON.stringify({}),
        cache: 'no-cache',
    });

    if (!response.ok) {
        reloadLoop();
        toastr.error('Settings could not be loaded after multiple attempts. Please try again later.');
        throw new Error('Error getting settings');
    }

    const data = await response.json();
    if (data.result != 'file not find' && data.settings) {
        settings = JSON.parse(data.settings);
        if (settings.username !== undefined && settings.username !== '') {
            name1 = settings.username;
            $('#your_name').val(name1);
        }

        await setUserControls(data.enable_accounts);

        // Allow subscribers to mutate settings
        eventSource.emit(event_types.SETTINGS_LOADED_BEFORE, settings);

        //Load KoboldAI settings
        koboldai_setting_names = data.koboldai_setting_names;
        koboldai_settings = data.koboldai_settings;
        koboldai_settings.forEach(function (item, i, arr) {
            koboldai_settings[i] = JSON.parse(item);
        });

        let arr_holder = {};

        $('#settings_preset').empty();
        $('#settings_preset').append(
            '<option value="gui">GUI KoboldAI Settings</option>',
        ); //adding in the GUI settings, since it is not loaded dynamically

        koboldai_setting_names.forEach(function (item, i, arr) {
            arr_holder[item] = i;
            $('#settings_preset').append(`<option value=${i}>${item}</option>`);
            //console.log('loading preset #'+i+' -- '+item);
        });
        koboldai_setting_names = {};
        koboldai_setting_names = arr_holder;
        preset_settings = settings.preset_settings;

        if (preset_settings == 'gui') {
            selectKoboldGuiPreset();
        } else {
            if (typeof koboldai_setting_names[preset_settings] !== 'undefined') {
                $(`#settings_preset option[value=${koboldai_setting_names[preset_settings]}]`)
                    .attr('selected', 'true');
            } else {
                preset_settings = 'gui';
                selectKoboldGuiPreset();
            }
        }

        novelai_setting_names = data.novelai_setting_names;
        novelai_settings = data.novelai_settings;
        novelai_settings.forEach(function (item, i, arr) {
            novelai_settings[i] = JSON.parse(item);
        });
        arr_holder = {};

        $('#settings_preset_novel').empty();

        novelai_setting_names.forEach(function (item, i, arr) {
            arr_holder[item] = i;
            $('#settings_preset_novel').append(`<option value=${i}>${item}</option>`);
        });
        novelai_setting_names = {};
        novelai_setting_names = arr_holder;

        //Load AI model config settings

        amount_gen = settings.amount_gen;
        if (settings.max_context !== undefined)
            max_context = parseInt(settings.max_context);

        swipes = settings.swipes !== undefined ? !!settings.swipes : true;  // enable swipes by default
        $('#swipes-checkbox').prop('checked', swipes); /// swipecode
        hideSwipeButtons();
        showSwipeButtons();

        // Kobold
        loadKoboldSettings(settings.kai_settings ?? settings);

        // Novel
        loadNovelSettings(settings.nai_settings ?? settings);
        $(`#settings_preset_novel option[value=${novelai_setting_names[nai_settings.preset_settings_novel]}]`).attr('selected', 'true');

        // TextGen
        loadTextGenSettings(data, settings);


        // OpenAI
        loadOpenAISettings(data, settings.oai_settings ?? settings);

        // Horde
        loadHordeSettings(settings);

        // Load power user settings
        await loadPowerUserSettings(settings, data);

        // Apply theme toggles from power user settings
        applyPowerUserSettings();

        // Load character tags
        loadTagsSettings(settings);

        // Load background
        loadBackgroundSettings(settings);

        // Load proxy presets
        loadProxyPresets(settings);

        // Allow subscribers to mutate settings
        eventSource.emit(event_types.SETTINGS_LOADED_AFTER, settings);

        // Set context size after loading power user (may override the max value)
        $('#max_context').val(max_context);
        $('#max_context_counter').val(max_context);

        $('#amount_gen').val(amount_gen);
        $('#amount_gen_counter').val(amount_gen);

        //Load which API we are using
        if (settings.main_api == undefined) {
            settings.main_api = 'kobold';
        }

        if (settings.main_api == 'poe') {
            settings.main_api = 'openai';
        }

        main_api = settings.main_api;
        $('#main_api').val(main_api);
        $('#main_api option[value=' + main_api + ']').attr(
            'selected',
            'true',
        );
        changeMainAPI();


        //Load User's Name and Avatar
        initUserAvatar(settings.user_avatar);
        setPersonaDescription();

        //Load the active character and group
        active_character = settings.active_character;
        active_group = settings.active_group;

        //Load the API server URL from settings
        api_server = settings.api_server;
        $('#api_url_text').val(api_server);

        setWorldInfoSettings(settings.world_info_settings ?? settings, data);

        selected_button = settings.selected_button;

        if (data.enable_extensions) {
            const enableAutoUpdate = Boolean(data.enable_extensions_auto_update);
            const isVersionChanged = settings.currentVersion !== currentVersion;
            await loadExtensionSettings(settings, isVersionChanged, enableAutoUpdate);
            await eventSource.emit(event_types.EXTENSION_SETTINGS_LOADED);
        }

        firstRun = !!settings.firstRun;

        if (firstRun) {
            hideLoader();
            await doOnboarding(user_avatar);
            firstRun = false;
        }
    }
    await validateDisabledSamplers();
    settingsReady = true;
    eventSource.emit(event_types.SETTINGS_LOADED);
}