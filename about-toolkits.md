## About Toolkits: How tools work, where they live, and how you customize them

### What is a Toolkit?
A toolkit is a typed, modular collection of tools (with Zod-validated inputs/outputs) that can be called by the LLM via tool-calling. Each toolkit ships both client and server definitions.

```21:36:src/toolkits/README.md
### Registration Process

After building your toolkit, you need to register it in three places:

1. Add your toolkit id to the enum in `shared.ts`
2. Import and add it to the `clientToolkits` object in `client.ts`
3. Import and add it to the `serverToolkits` object in `server.ts`
```

```23:36:src/toolkits/README.md
The toolkit system uses a **client/server separation** architecture where each toolkit has both client-side and server-side implementations:

```

```54:68:src/toolkits/README.md
#### `types.ts` - Type Definitions
- `BaseTool` ...
- `ClientTool` ...
- `ServerTool` ...
- `ClientToolkit` & `ServerToolkit` ...

#### `create-toolkit.ts` - Toolkit Factories
- `createClientToolkit()`
- `createServerToolkit()`
```

### Where toolkits and tools are stored
- Client registry (drives UI):

```28:41:src/toolkits/toolkits/client.ts
export const clientToolkits: ClientToolkits = {
  [Toolkits.E2B]: e2bClientToolkit,
  [Toolkits.Memory]: mem0ClientToolkit,
  [Toolkits.Image]: imageClientToolkit,
  [Toolkits.Exa]: exaClientToolkit,
  [Toolkits.Github]: githubClientToolkit,
  [Toolkits.GoogleCalendar]: googleCalendarClientToolkit,
  [Toolkits.Notion]: notionClientToolkit,
  [Toolkits.GoogleDrive]: googleDriveClientToolkit,
  [Toolkits.Discord]: discordClientToolkit,
  [Toolkits.Strava]: stravaClientToolkit,
  [Toolkits.Spotify]: spotifyClientToolkit,
  [Toolkits.Video]: videoClientToolkit,
};
```

- Server registry (exposes callable tools to the model):

```27:40:src/toolkits/toolkits/server.ts
export const serverToolkits: ServerToolkits = {
  [Toolkits.Exa]: exaToolkitServer,
  [Toolkits.Image]: imageToolkitServer,
  [Toolkits.Github]: githubToolkitServer,
  [Toolkits.GoogleCalendar]: googleCalendarToolkitServer,
  [Toolkits.GoogleDrive]: googleDriveToolkitServer,
  [Toolkits.Memory]: mem0ToolkitServer,
  [Toolkits.Notion]: notionToolkitServer,
  [Toolkits.E2B]: e2bToolkitServer,
  [Toolkits.Discord]: discordToolkitServer,
  [Toolkits.Strava]: stravaToolkitServer,
  [Toolkits.Spotify]: spotifyToolkitServer,
  [Toolkits.Video]: videoToolkitServer,
};
```

- Toolkit IDs and type mapping live here:

```26:39:src/toolkits/toolkits/shared.ts
export enum Toolkits {
  Exa = "exa",
  Image = "image",
  Github = "github",
  GoogleCalendar = "google-calendar",
  GoogleDrive = "google-drive",
  Memory = "memory",
  Notion = "notion",
  E2B = "e2b",
  Discord = "discord",
  Strava = "strava",
  Spotify = "spotify",
  Video = "video",
}
```

### How a tool is defined
- Base tool contract uses Zod schemas and a description.

```75:82:src/toolkits/types.ts
export type BaseTool<Args extends ZodRawShape = ZodRawShape, Result extends ZodRawShape = ZodRawShape> = {
  description: string;
  inputSchema: ZodObject<Args>;
  outputSchema: ZodObject<Result>;
};
```

- Client-facing toolkits add UI metadata: name, description, icon, env var requirements, optional configuration form, and grouping.

```39:53:src/toolkits/types.ts
export type ClientToolkitConifg<Parameters extends ZodRawShape = ZodRawShape> = {
  name: string;
  description: string;
  icon: React.FC<{ className?: string }>;
  envVars: EnvVars;
  form: React.ComponentType<{ parameters: z.infer<ZodObject<Parameters>>; setParameters: (parameters: z.infer<ZodObject<Parameters>>) => void; }> | null;
  type: ToolkitGroups;
  Wrapper?: React.FC<{ Item: React.FC<{ isLoading: boolean; onSelect?: () => void }> }>;
};
```

- Factories compose tool definitions into client/server toolkits.

```14:36:src/toolkits/create-toolkit.ts
export const createClientToolkit = <...> => {
  return { ...toolkitConfig, ...clientToolkitConfig, tools: /* base + client tool configs */ };
};
```

```39:66:src/toolkits/create-toolkit.ts
export const createServerToolkit = <...> => {
  return { systemPrompt, tools: async (params) => /* base + server tool configs */ };
};
```

### How tools show up in the UI and can be customized
- The “Add Toolkits” button opens a selector where you can enable/disable toolkits and optionally configure parameters via a form.

```71:104:src/app/(general)/_components/chat/input/tools.tsx
return (
  <ToolkitSelect
    isOpen={isOpen}
    onOpenChange={setIsOpen}
    toolkits={toolkits}
    addToolkit={addToolkit}
    removeToolkit={removeToolkit}
    workbench={workbench}
  >
    <Button ...>
      {toolkits.length > 0 ? <ToolkitIcons .../> : <Wrench />}
      <span className="hidden md:block">{toolkits.length > 0 ? `${toolkits.length} Toolkit${toolkits.length > 1 ? "s" : ""}` : "Add Toolkits"}</span>
    </Button>
  </ToolkitSelect>
);
```

- The selector lists enabled and available toolkits, supports search, and can auto-enable toolkits via URL query params.

```31:61:src/components/toolkit/toolkit-list/index.tsx
useEffect(() => {
  const updatedToolkits = Object.entries(clientToolkits).filter(([id]) => {
    return (
      searchParams.get(id) === "true" &&
      !selectedToolkits.some((t) => t.id === (id as Toolkits))
    );
  });
  if (updatedToolkits.length > 0) {
    updatedToolkits.forEach(([id, toolkit]) => {
      onAddToolkit({ id: id as Toolkits, parameters: {}, toolkit: toolkit as ClientToolkit });
    });
    window.history.replaceState({}, "", pathname);
  }
}, ...);
```

- Each toolkit may require environment variables. In development, a dialog guides you to set missing vars before enabling.

```91:112:src/components/toolkit/toolkit-list/item.tsx
const missingEnvVars = useToolkitMissingEnvVars(toolkit);
return (
  <>
    <CommandItem ... onSelect={missingEnvVars.length > 0 ? () => setIsOpen(true) : onSelect} />
    <EnvVarDialog isOpen={isOpen} onOpenChange={setIsOpen} resourceName={`the ${toolkit.name} toolkit`} envVars={missingEnvVars} onSuccess={onSelect} />
  </>
);
```

```34:58:src/components/env-vars/env-var-dialog.tsx
<Dialog open={isOpen} onOpenChange={onOpenChange}>
  <DialogContent className="gap-4" showCloseButton={false}>
    <DialogHeader>
      <DialogTitle>Insufficient Env Vars</DialogTitle>
      <DialogDescription>
        In order to use {resourceName}, you will need the following environment variables:
      </DialogDescription>
    </DialogHeader>
    <EnvVarForm ... />
  </DialogContent>
</Dialog>
```

- If a toolkit has parameters, a modal form is presented before enabling; inputs are validated against the toolkit’s Zod schema.

```125:174:src/components/toolkit/toolkit-list/item.tsx
{toolkit.form && (
  <toolkit.form parameters={parameters} setParameters={setParameters} />
)}
<Button onClick={() => addToolkit({ id, toolkit, parameters })} disabled={!toolkit.parameters.safeParse(parameters).success}>
  Enable
</Button>
```

### How selections are persisted and restored
- Selections are saved to cookies client-side, and read back server-side as initial preferences for the chat session.

```197:204:src/app/(general)/_contexts/chat-context.tsx
const setToolkits = (newToolkits: Array<SelectedToolkit>) => {
  setToolkitsState(newToolkits);
  clientCookieUtils.setToolkits(newToolkits);
};
```

```28:41:src/lib/cookies/client.ts
setToolkits(toolkits: Array<{ id: string; toolkit: ClientToolkit; parameters: z.infer<ClientToolkit["parameters"]>; }>): void {
  const persistedToolkits = toolkits.map((t) => ({ id: t.id, parameters: t.parameters }));
  setCookie(COOKIE_KEYS.TOOLKITS, persistedToolkits);
}
```

```18:39:src/lib/cookies/server.ts
export const serverCookieUtils = {
  async getPreferences(): Promise<ChatPreferences> {
    const cookieStore = await cookies();
    return {
      selectedChatModel: safeParseJson(cookieStore.get(COOKIE_KEYS.SELECTED_CHAT_MODEL)?.value, undefined),
      imageGenerationModel: safeParseJson(cookieStore.get(COOKIE_KEYS.IMAGE_GENERATION_MODEL)?.value, undefined),
      useNativeSearch: safeParseJson(cookieStore.get(COOKIE_KEYS.USE_NATIVE_SEARCH)?.value, false),
      toolkits: safeParseJson(cookieStore.get(COOKIE_KEYS.TOOLKITS)?.value, []),
    };
  },
};
```

- Workbenches can store a default set of toolkit IDs; the selector allows saving updates to the workbench.

```104:147:src/components/toolkit/toolkit-select.tsx
const { mutate: updateWorkbench } = api.workbenches.updateWorkbench.useMutation({...});
<Button onClick={handleSave}>
  Update Workbench
</Button>
```

### How tools are actually called by the LLM
- At request time, the server resolves each selected toolkit to its server tools and passes them into the AI SDK as callable tools. The model can then invoke them (tool-calling), with inputs validated by the tool’s Zod schema.

```157:171:src/app/api/chat/route.ts
const toolkitTools = await Promise.all(
  toolkits.map(async ({ id, parameters }) => {
    const toolkit = getServerToolkit(id);
    const tools = await toolkit.tools(parameters);
    return Object.keys(tools).reduce((acc, toolName) => {
      const serverTool = tools[toolName as keyof typeof tools];
      acc[`${id}_${toolName}`] = tool({ description: serverTool.description, parameters: serverTool.inputSchema, execute: async (args) => {/* ... */} });
      return acc;
    }, {} as Record<string, Tool>);
  }),
);
```

---

### TL;DR
- Toolkits are typed bundles of tools with client (UI, env vars, optional form) and server (callbacks) parts.
- They’re registered in `clientToolkits` and `serverToolkits` and referenced by ID from `Toolkits`.
- Users enable/disable and configure toolkits in the UI; missing env vars are surfaced via a dialog.
- Selections persist via cookies and can be pre-enabled via URL params or saved to workbenches.
- On send, the server supplies selected tools to the model; the model invokes them via tool-calling with schema validation.

### Custom in-chat tool UI
- Each tool provides custom UI via two client renderers in its client config:
  - CallComponent: rendered during call/partial-call
  - ResultComponent: rendered when the tool returns

```99:112:src/toolkits/types.ts
export type ClientToolConfig<
  Args extends ZodRawShape = ZodRawShape,
  Result extends ZodRawShape = ZodRawShape,
> = {
  CallComponent: React.ComponentType<{
    args: DeepPartial<z.infer<ZodObject<Args>>>;
    isPartial: boolean;
  }>;
  ResultComponent: React.ComponentType<{
    args: z.infer<ZodObject<Args>>;
    result: z.infer<ZodObject<Result>>;
    append: (message: CreateMessage) => void;
  }>;
};
```

- The message renderer automatically discovers the correct client toolkit and renders your UI components for each tool invocation.

```47:59:src/app/(general)/_components/chat/messages/message-tool.tsx
const [server, tool] = toolName.split("_");
const typedServer = server as Toolkits;
const clientToolkit = getClientToolkit(typedServer);
const typedTool = tool as ServerToolkitNames[typeof typedServer];
const toolConfig = clientToolkit.tools[typedTool];
```

```131:140:src/app/(general)/_components/chat/messages/message-tool.tsx
{toolInvocation.args && (
  <toolConfig.CallComponent
    args={
      toolInvocation.args as DeepPartial<
        z.infer<typeof toolConfig.inputSchema>
      >
    }
    isPartial={toolInvocation.state === "partial-call"}
  />
)}
```

```173:186:src/app/(general)/_components/chat/messages/message-tool.tsx
<MessageToolResultComponent
  Component={({ append }) => (
    <toolConfig.ResultComponent
      args={
        toolInvocation.args as z.infer<
          typeof toolConfig.inputSchema
        >
      }
      result={result.result}
      append={append}
    />
  )}
/>
```

- Example: E2B Code Interpreter tool defines rich Call/Result UIs for code, images, HTML/SVG, JSON, charts, and logs.

```152:165:src/toolkits/toolkits/e2b/tools/run_code/client.tsx
export const e2bRunCodeToolConfigClient: ClientToolConfig<
  typeof baseRunCodeTool.inputSchema.shape,
  typeof baseRunCodeTool.outputSchema.shape
> = {
  CallComponent: ({ args, isPartial }) => {
    return (
      <div className="w-full space-y-2">
        <h1 className="text-muted-foreground text-sm font-medium">
          {isPartial ? "Writing Python Code" : "Executing Python Code"}
        </h1>
        {args.code && <CodeBlock language="python" value={args.code} />}
      </div>
    );
  },
```

```166:176:src/toolkits/toolkits/e2b/tools/run_code/client.tsx
ResultComponent: ({ result, args: { code } }) => {
  const hasResults = result.results && result.results.length > 0;
  const hasLogs =
    result.logs.stdout.length > 0 || result.logs.stderr.length > 0;

  return (
    <div className="space-y-2">
      <Accordion type="single" collapsible>
        <AccordionItem value="args">
          <AccordionTrigger className="cursor-pointer p-0 hover:no-underline">
            <h2 className="text-muted-foreground text-sm font-medium">
              Code
            </h2>
```

### Running scripts in a sandbox
- The E2B toolkit provides a secure Python sandbox using @e2b/code-interpreter. The server creates a sandbox, runs user-provided code, returns results/logs, then tears down the sandbox.

```18:28:src/toolkits/toolkits/e2b/tools/run_code/server.ts
const sandbox = await Sandbox.create({
  apiKey: process.env.E2B_API_KEY,
});
const { results, logs } = await sandbox.runCode(code);
await sandbox.kill();
return { results: results, logs: logs };
```

- It requires the E2B_API_KEY; ensure this is set in your environment.

```11:16:src/toolkits/toolkits/e2b/tools/run_code/server.ts
if (!env.E2B_API_KEY) {
  throw new Error(
    "E2B_API_KEY environment variable is required but not set",
  );
}
```

```60:61:src/env.js
// Code Interpreter toolkit
if (process.env.E2B_API_KEY) toolkitsSchema.E2B_API_KEY = z.string();
```

- Toolkit descriptor highlights the capability:

```6:13:src/toolkits/toolkits/e2b/server.ts
export const e2bToolkitServer = createServerToolkit(
  baseE2BToolkitConfig,
  `You have access to the E2B toolkit for secure code execution and development environments. This toolkit provides:

- **Run Code**: Execute Python code in isolated, secure cloud environments.`,
  async () => ({
    [E2BTools.RunCode]: e2bRunCodeToolConfigServer(),
  }),
);
```
