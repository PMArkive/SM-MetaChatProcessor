# Meta Chat Processor

This plugin is intended to merge, replace and extend some previous chat processors.
With the difference to other chat processors, that I want to keep compatibility for plugins that depend on these chat processors to some degree.

Differences: Instead of a simple forward, there's 3.5 stacked forwards for pre, early, normal and late manipulation, followed by a 'color', and 'formatted' forward if the message was changed before the post forward.
This should offer enough flexibility to implement compatibility for other chat processors and then some.
Supporting certain chat processors requires basic name tagging capabilities. In MCP this is done through name prefixes, instead of tags. The prefix merges tag color, tag and name color.
The message format in MCP is broken down more than usual, allowing for a more refined manipulation, and registration of custom prefixes, e.g. (Region)name : message.
Lastly the message is post processed to remove redundant colors and subsequently save bytes in case two plugins can't agree on where to put colors.
The last difference is probably that MCP uses private forwards instead of global forwards, meaning you have to register your functions like e.g. Events or SDKHooks.

In order to implement some of these features, the available data has to be expanded. This is done with a gamedata file on one side, and translation files on the other.
The gamedata file can handle team colors, while the translation file handles things like default colors, team names, ect. Note that games that use (TEAM) as prefix can
be forced to use the actual team name and vice versa, thus a translation file per game is required. To keep the localized nature, custom senderflag and group names have
to be registered with translation phrases, that can then be manipulated numerically.

On PrintToChat support
While CPrintToChat from the color includes uses SayText2 to send messages, they are marked as non-hookable so only regular PrintToChat messages could be hooked.
In addition to that, these messages might already be sent on a per-client basis for translation or otherwise, making parsing very hard!
Instead of doing the impossible, MCP instead has a native to send SayText2 messages, to basically fake say messages.

Time estimation:
The profiler timed about .035 ms for pre and about .06 ms post processing with one player on the server (that is with SCP compat and CCC formatting the name).
With a 32 slot TF2 server at 15 mspt my wort case estimation is roughly 13% of a game tick.
Without baseline and chat messages not happening every game tick I'd say this is not hyper speed, but acceptable.

## Config & Setup

As mentioned above, MCP implements compatibility layers for Simple Chat-Processor, Drixevel's Chat-Processor and Cider Chat-Processor. As I expect most people to not read the docs or just skim over them, all three compat layers are enabled by default.
I want to emphasise here that I am only implementing API compatibility, not feature pairity! In addition you can switch the transport method from using SayText2 packets to TextMsg packets (system/plugin messages).
Simple Chat-Processor also had the quirk that the Post call was only called if the message was changed. I have an optional fix for that in place, that you can enable in the config as well.
The config can be found at `addons/sourcemod/config/metachatprocessor.cfg`:
```json
"config"
{
	"Compatibility"
	{
		"SCP Redux"		"1"
		"Drixevel"		"1"
		"Cider"			"1"
		"Fix Post Calls" "0"
	}
	"Transport"		"SayText"
}
```

## Forwards & Call order

#### mcpHookPre:
`Action (int& sender, ArrayList recipients, mcpSenderFlag& senderflags, mcpTargetGroup& targetgroup, mcpMessageOption& options, char[] targetgroupColor)`

Called before the usual processing for early blocking/management.

#### mcpHookEarly, mcpHookDefault, mcpHookLate:
`Action (int& sender, ArrayList recipients, mcpSenderFlag& senderflags, mcpTargetGroup& targetgroup, mcpMessageOption& options, char[] targetgroupColor, char[] name, char[] message)`

These are the main forwards, please use mcpHookDefault unless there's a conflict and it would break.

#### mcpHookColors:
`Action (int sender, mcpSenderFlag senderflags, mcpTargetGroup targetgroup, mcpMessageOption options, char[] nameTag, char[] displayName, char[] chatColor)`

The dedicated forward for when mcp prefix and colors are applied. the display name might have formats from earlier forwards, nameTag has the tag&color from mcp.

#### mcpHookFormatted:
`Action (int sender, int recipient, mcpSenderFlag senderflags, mcpTargetGroup targetgroup, mcpMessageOption options, char[] formatted)`

Called after the client specific format translations are applied and the message is about to be sent to a client.

#### mcpHookPost:
`void (int sender, ArrayList recipients, mcpSenderFlag senderflags, mcpTargetGroup targetgroup, mcpMessageOption options, const char[] targetgroupColor, const char[] name, const char[] message)`

The message is sent and you may do some post sending cleanup.

## Other natives
#### Manage Senderflags:
With `MCP_RegisterSenderFlag` and `MCP_UnregisterSenderFlags` you can register custom sender flags.
This is done by translation phrase and will return a flag bit. Since theres 32 bits in a cell, there's a global limit of 32 senderflag. These will be concatinated within asterisks in front of a chat message (default flags are `*DEAD*` and `*SPEC*`).

#### Manage Targetgroup:
Using `MCP_RegisterTargetGroup` and `MCP_UnregisterTargetGroups` you can add custom message target group names.
Again, these use translation phrases but as only one target group can be used at a time, there can be almost any amount of target groups. Target groups are formatted between sender flags and username (default groups are `(TEAM)` or `(Spectator)`).
Through the modular translation system you can exchange the `(TEAM)` prefix for named team prefixes like `(Terrorists)` or `(Survivors)` and vice versa.
These groups are put into the enum in sequence, so checking for a team message can be done with this condition: `(mcpTargetTeam1 <= targetgroup <= mcpTargetTeamSender)`.

#### Manually Sending messages:
You can bypass the SayText2 hook by calling `MCP_SendChat` directly. This allows you to easily create messages outside the normal format specifications.

#### Escaping colors:
If you want to apply colors by color tags, as we are pretty much used to now, you might run into troubles when a client inputs curly braces / color codes in the input.
To prevent those from parsing, you can use `MCP_EscapeCurlies` and `MCP_UnecapeCurlies` which uses `MCP_PUA_ESCAPED_LCURLY` (\uEC01) as temporary replacement.
This character is from the private use block and should neither break anything nor render in the client. Since I can not predict how plugins will use curlies I cannot default replace them, in neither input nor color tags.

#### Manipulating recipients:
In order to help you manage the recipients list, there the `MCP_FindClients*` group of methods as well as `MCP_RemoveListElements`.
You should not worry about duplicate entries in the recipients list, that is already handled by MCP after each forward is called.

## About buffers
The message buffers in MCP are a bit bigger then previously. This is mostly to give color tags some additional space as they might collapse to no more than 7 bytes when parsed.
Please keep in mind that the maximum length for these network packages is around 256 bytes so you should not exceed `MCP_MAXLENGTH_MESSAGE`.


