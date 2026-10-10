
## _LazyPig.toc
```toc
## Interface: 11200
## Title: [|cff3fcf26K|r] LazyPig
## Version: 6.2.2
## Notes: Common tasks automation, type /lp for configuration.
## Author: Ogrisch, mrmr, |cffbe5effLexie|r and |c013220ffTrumpetX|r
## SavedVariables: LPCONFIG

LazyPig.xml
```

## Bindings.xml
```xml
<Bindings>
	<Binding name="LOGOUT" header="LP_HEADER">
		Logout()
	</Binding>
	<Binding name="UNSTUCK">
		Stuck()
	</Binding>
	<Binding name="RELOAD">
		ReloadUI()
	</Binding>
	<Binding name="DUEL">
		LazyPig_Duel_EFC()
	</Binding>
	<Binding name="WSGDROP">
		LazyPig_DropWSGFlag_NoggBuff()
	</Binding>
	<Binding name="MENU">
		LazyPig_Command()
	</Binding>
</Bindings>
```

## LazyPig.lua
```lua
local _G = _G or getfenv(0)

-- Default SavedVariables
LPCONFIG = {}
LPCONFIG.DISMOUNT = true           -- Auto Dismount
LPCONFIG.AUTOSTANCE = true         -- Auto Stance
LPCONFIG.CAM = false               -- Extended camera distance
LPCONFIG.GINV = true               -- Auto accept invites from guild members
LPCONFIG.FINV = true               -- Auto accept invites from friends
LPCONFIG.SINV = false              -- Auto accept invites from strangers
LPCONFIG.DINV = true               -- Disable auto accept invite whiel in bg or in bg queue
LPCONFIG.SUMM = false              -- Auto accept summons
LPCONFIG.EBG = true                -- Auto join battleground
LPCONFIG.LBG = true                -- Auto leave battleground
LPCONFIG.QBG = true                -- Auto queue battleground
LPCONFIG.RBG = true                -- Auto release spirit in battleground
LPCONFIG.SBG = false               -- Auto decline quest sharing while in battleground
LPCONFIG.AQUE = false              -- Announce when queueing for battleground as party leader
LPCONFIG.LOOT = false              -- Position loot frame at cursor
LPCONFIG.RIGHT = true              -- Improved right click
LPCONFIG.GREEN = nil               -- [number or nil] Auto roll on green items
LPCONFIG.ZG = 1                    -- [number or nil] ZG coins/bijou auto roll
LPCONFIG.MC = nil                  -- [number or nil] MC mats auto roll
LPCONFIG.AQ = nil                  -- [number or nil] AQ scarabs/idols auto roll
LPCONFIG.SAND = 1                  -- [number or nil] Corrupted sand auto roll
LPCONFIG.ES_SHARDS = nil           -- [number or nil] Dream Shrads auto roll
LPCONFIG.NAXX = nil                -- [number or nil] Scraps auto roll
LPCONFIG.ROLLMSG = false           -- Lazy Pig Auto Roll Messages
LPCONFIG.DUEL = false              -- Auto cancel duels
LPCONFIG.SPECIALKEY = false        -- Special key combinations
LPCONFIG.WORLDDUNGEON = false      -- Mute Wolrd chat while in dungeons
LPCONFIG.WORLDRAID = false         -- Mute Wolrd chat while in raid
LPCONFIG.WORLDBG = false           -- Mute Wolrd chat while in battleground
LPCONFIG.WORLDUNCHECK = false      -- Mute Wolrd chat always
LPCONFIG.SPAM = false              -- Hide players spam messages
LPCONFIG.SPAM_UNCOMMON = false     -- Hide green items roll messages
LPCONFIG.SPAM_RARE = false         -- Hide blue items roll messages
LPCONFIG.SHIFTSPLIT = false        -- Improved stack splitting with shift
LPCONFIG.REZ = false               -- Auto accept resurrection while in raid, dungeon or bg if resurrecter is out of combat
LPCONFIG.GOSSIP = true             -- Auto proccess gossip
LPCONFIG.SALVA = nil               -- [number or nil] Autoremove Blessing of Salvation
LPCONFIG.REMOVEMANABUFFS = false   -- Autoremove Blessing of Wisdom, Arcane Intellect, Prayer of Spirit

BINDING_HEADER_LP_HEADER = "_LazyPig";
BINDING_NAME_LOGOUT = "Logout";
BINDING_NAME_UNSTUCK = "Unstuck";
BINDING_NAME_RELOAD = "Reaload UI";
BINDING_NAME_DUEL = "Target WSG EFC/Duel Request-Cancel";
BINDING_NAME_WSGDROP = "Drop WSG Flag/Remove Slow Fall";
BINDING_NAME_MENU = "_LazyPig Menu";

local Original_SelectGossipActiveQuest = SelectGossipActiveQuest;
local Original_SelectGossipAvailableQuest = SelectGossipAvailableQuest;
local Original_SelectActiveQuest = SelectActiveQuest;
local Original_SelectAvailableQuest = SelectAvailableQuest;
local OriginalLootFrame_OnEvent = LootFrame_OnEvent;
local OriginalLootFrame_Update = LootFrame_Update;
local OriginalUseContainerItem = UseContainerItem;
local Original_ChatFrame_OnEvent = ChatFrame_OnEvent;
local Original_StaticPopup_OnShow = StaticPopup_OnShow;
local Original_QuestRewardItem_OnClick = QuestRewardItem_OnClick

local roster_task_refresh = 0
local last_click = 0
local delayaction = 0
local tradedelay = 0
local bgstatus = 0
local tmp_splitval = 1
local passpopup = 0
local timeLeft = 0
local updateInterval = 0.1
local ctrltime = 0
local alttime = 0
local shift_time = 0
local ctrlalttime = 0
local ctrlshifttime = 0
local altshifttime = 0
local greenrolltime = 0
local shamanTankTree = 2 -- Enhancement
local shamanTankTalent = 11 -- Spirit Armor

local timer_split = nil
local player_summon_confirm = nil
local player_summon_message = nil
local player_bg_confirm = nil
local player_bg_message = nil
local afk_active = nil
local duel_active = nil
local merchantstatus = nil
local tradestatus = nil
local mailstatus = nil
local auctionstatus = nil
local auctionbrowse = nil
local bankstatus = nil
local channelstatus = nil
local battleframe = nil
local wsgefc = nil

local ScheduleButton = {}
local ScheduleFunction = {}
local QuestRecord = {}
local ActiveQuest = {}
local AvailableQuest = {}
local ChatMessage = {{}, {}, INDEX = 1}
local ScheduleSplit = {}
local ScheduleSplitCount = {}
local GossipOptions = {}

ScheduleSplit.active = nil
ScheduleSplit.sslot = {}
ScheduleSplit.dbag = {}
ScheduleSplit.dslot = {}
ScheduleSplit.sbag = {}
ScheduleSplit.count = {}

local function twipe(t)
	if type(t) ~= "table" then return {} end
	for i = table.getn(t), 1, -1 do table.remove(t, i) end
	for k in pairs(t) do t[k] = nil end
	return t
end

function LazyPig_OnLoad()
	SelectGossipActiveQuest = LazyPig_SelectGossipActiveQuest;
	SelectGossipAvailableQuest = LazyPig_SelectGossipAvailableQuest;
	SelectActiveQuest = LazyPig_SelectActiveQuest;
	SelectAvailableQuest = LazyPig_SelectAvailableQuest;
	LootFrame_OnEvent = LazyPig_LootFrame_OnEvent;
	LootFrame_Update = LazyPig_LootFrame_Update;
	UseContainerItem = LazyPig_UseContainerItem;
	ChatFrame_OnEvent = LazyPig_ChatFrame_OnEvent;
	StaticPopup_OnShow = LazyPig_StaticPopup_OnShow;
	QuestRewardItem_OnClick = LazyPig_QuestRewardItem_OnClick

	SLASH_LAZYPIG1 = "/lp";
	SLASH_LAZYPIG2 = "/lazypig";
	SlashCmdList["LAZYPIG"] = LazyPig_Command;
	this:RegisterEvent("ADDON_LOADED")
	this:RegisterEvent("PLAYER_LOGIN")
	this:RegisterEvent("CHAT_MSG")
	this:RegisterEvent("CHAT_MSG_SYSTEM")
	this:RegisterEvent("PARTY_INVITE_REQUEST")
	this:RegisterEvent("CONFIRM_SUMMON")
	this:RegisterEvent("RESURRECT_REQUEST")
	this:RegisterEvent("UPDATE_BATTLEFIELD_STATUS")
	this:RegisterEvent("CHAT_MSG_BG_SYSTEM_ALLIANCE")
	this:RegisterEvent("CHAT_MSG_BG_SYSTEM_HORDE")
	this:RegisterEvent("CHAT_MSG_BG_SYSTEM_NEUTRAL")
	this:RegisterEvent("BATTLEFIELDS_SHOW")
	this:RegisterEvent("GOSSIP_SHOW")
	this:RegisterEvent("QUEST_GREETING")
	this:RegisterEvent("UI_ERROR_MESSAGE")
	this:RegisterEvent("QUEST_PROGRESS")
	this:RegisterEvent("QUEST_COMPLETE")
	this:RegisterEvent("START_LOOT_ROLL")
	this:RegisterEvent("DUEL_REQUESTED")
	this:RegisterEvent("MERCHANT_SHOW")
	this:RegisterEvent("MERCHANT_CLOSED")
	this:RegisterEvent("TRADE_SHOW")
	this:RegisterEvent("TRADE_CLOSED")
	this:RegisterEvent("MAIL_SHOW")
	this:RegisterEvent("MAIL_CLOSED")
	this:RegisterEvent("AUCTION_HOUSE_SHOW")
	this:RegisterEvent("AUCTION_HOUSE_CLOSED")
	this:RegisterEvent("BANKFRAME_OPENED")
	this:RegisterEvent("BANKFRAME_CLOSED")
	this:RegisterEvent("ZONE_CHANGED_NEW_AREA")
	this:RegisterEvent("PLAYER_UNGHOST")
	this:RegisterEvent("PLAYER_DEAD")
	this:RegisterEvent("PLAYER_AURAS_CHANGED")
	this:RegisterEvent("UPDATE_BONUS_ACTIONBAR")
	this:RegisterEvent("UNIT_INVENTORY_CHANGED")
	this:RegisterEvent("UI_INFO_MESSAGE")
end

function LazyPig_Command()
	if LazyPigOptionsFrame:IsShown() then
		LazyPigOptionsFrame:Hide()
	else
		LazyPigOptionsFrame:Show()
	end
end

function LazyPig_OnUpdate(elapsed)
	timeLeft = timeLeft - elapsed
	if timeLeft > 0 then return end
	timeLeft = updateInterval

	local current_time = GetTime();
	local shiftstatus = IsShiftKeyDown();
	local ctrlstatus = IsControlKeyDown();
	local altstatus = IsAltKeyDown();

	if shiftstatus then
		shift_time = current_time
	elseif altstatus and not ctrlstatus and current_time > alttime then
		alttime = current_time + 0.75
	elseif not altstatus and ctrlstatus and current_time > ctrltime then
		ctrltime = current_time + 0.75
	elseif not altstatus and not ctrlstatus or altstatus and ctrlstatus then
		ctrltime = 0
		alttime = 0
	end
	if ctrlstatus and not shiftstatus and altstatus and current_time > ctrlalttime then
		ctrlalttime = current_time + 0.75
	elseif ctrlstatus and shiftstatus and not altstatus and current_time > ctrlshifttime then
		ctrlshifttime = current_time + 0.75
	elseif not ctrlstatus and shiftstatus and altstatus and current_time > altshifttime then
		altshifttime = current_time + 0.75
	elseif ctrlstatus and shiftstatus and altstatus then
		ctrlshifttime = 0
		ctrlalttime = 0
		altshifttime = 0
	end

	if shift_time == current_time  then
		if not (UnitExists("target") and UnitIsUnit("player", "target")) then
			--
		elseif not battleframe then
			battleframe = current_time
		elseif (current_time - battleframe) > 3 then
			--BattlefieldFrame:Show()
			battleframe = current_time
		end
	elseif battleframe then
		battleframe = nil
	end

	if LPCONFIG.SPECIALKEY then
		if ctrlstatus and shiftstatus and altstatus and current_time > delayaction then
			delayaction = current_time + 1
			local bind1, bind2 =  GetBindingKey("LOGOUT")
			if not bind1 and not bind2 then
				Logout();
			end
		elseif ctrlstatus and not shiftstatus and altstatus and not auctionstatus and not mailstatus and current_time > delayaction then
			if tradestatus then
				AcceptTrade();
			elseif not tradestatus and UnitExists("target") and UnitIsPlayer("target") and UnitIsFriend("target", "player") and not UnitIsUnit("player", "target") and CheckInteractDistance("target", 2) and (current_time + 0.25) > ctrlalttime and current_time > tradedelay then
				InitiateTrade("target");
				delayaction = current_time + 2
			end
		elseif ctrlstatus and shiftstatus and not altstatus and UnitIsPlayer("target") and UnitIsFriend("target", "player") and current_time > delayaction and (current_time + 0.25) > ctrlshifttime then
			delayaction = current_time + 1.5
			FollowUnit("target");
		elseif not ctrlstatus and shiftstatus and altstatus and UnitIsPlayer("target") and current_time > delayaction and (current_time + 0.25) > altshifttime then
			delayaction = current_time + 1.5
			InspectUnit("target");
		end

		if ctrlstatus and not shiftstatus and altstatus or passpopup > current_time then
			if current_time > delayaction and not LazyPig_BindLootOpen() and not LazyPig_RollLootOpen() and LazyPig_GreenRoll() then
				delayaction = current_time + 1
			elseif current_time > delayaction then
				for i=1,STATICPOPUP_NUMDIALOGS do
					local frame = _G["StaticPopup"..i]
					if frame:IsShown() then
						--DEFAULT_CHAT_FRAME:AddMessage(frame.which)
						if frame.which == "DEATH" and HasSoulstone() then
							_G["StaticPopup"..i.."Button2"]:Click();
							if passpopup < current_time then delayaction = current_time + 0.5 end
						elseif frame.which ~= "CONFIRM_SUMMON" and frame.which ~= "CONFIRM_BATTLEFIELD_ENTRY" and frame.which ~= "CAMP" and frame.which ~= "AREA_SPIRIT_HEAL"  then --and release and

							_G["StaticPopup"..i.."Button1"]:Click();
							if passpopup < current_time then delayaction = current_time + 0.5 end
						end
					end
				end
			end
		end

		if ctrlstatus and not shiftstatus and altstatus then
			if current_time > delayaction then
				if auctionstatus and AuctionFrameAuctions and AuctionFrameAuctions:IsVisible() and AuctionsCreateAuctionButton then
					ScheduleButtonClick(AuctionsCreateAuctionButton, 0);
				elseif auctionstatus and AuctionFrameBrowse and AuctionFrameBrowse:IsVisible() and BrowseBuyoutButton then
					ScheduleButtonClick(BrowseBuyoutButton, 0.55);
				elseif CT_MailFrame and CT_MailFrame:IsVisible() and CT_MailFrame.num > 0 and strlen(CT_MailNameEditBox:GetText()) > 0 and CT_Mail_AcceptSendFrameSendButton then
					ScheduleButtonClick(CT_Mail_AcceptSendFrameSendButton, 1.25);
				elseif GMailFrame and GMailFrame:IsVisible() and GMailFrame.num > 0 and strlen(GMailSubjectEditBox:GetText()) > 0 and GMailAcceptSendFrameSendButton then
					ScheduleButtonClick(GMailAcceptSendFrameSendButton, 1.25);
				elseif mailstatus and SendMailFrame and SendMailFrame:IsVisible() and SendMailMailButton then
					ScheduleButtonClick(SendMailMailButton, 0);
				elseif mailstatus and OpenMailFrame and OpenMailFrame:IsVisible() then
					if OpenMailFrame.money and OpenMailMoneyButton then
						ScheduleButtonClick(OpenMailMoneyButton, 0);
					elseif OpenMailPackageButton then
						ScheduleButtonClick(OpenMailPackageButton, 0);
					end
				end
			end
			LazyPig_AutoLeaveBG();
		elseif not ctrlstatus and shiftstatus and altstatus and current_time > delayaction then
			if auctionstatus and AuctionFrameBrowse and AuctionFrameBrowse:IsVisible() and BrowseBidButton then
				ScheduleButtonClick(BrowseBidButton, 0.55);
			end
		end
	end

	if merchantstatus and altstatus and current_time > last_click and not CursorHasItem() then
		last_click = current_time + 0.25
		LazyPig_GreySellRepair();
	end

	if not QuestHaste then
		if altstatus then
			if QuestFrameDetailPanel:IsVisible() then
				AcceptQuest();
			end
		elseif QuestRecord["details"] then
			LazyPig_RecordQuest();
		end
	end

	if not afk_active and player_bg_confirm then
		Check_Bg_Status();
	end

	if bgstatus ~= 0 and (bgstatus + 0.5) > current_time then
		bgstatus = 0
		Check_Bg_Status()
		LazyPig_AutoLeaveBG()
	end

	if(current_time - roster_task_refresh) > 29 then
		roster_task_refresh = current_time
		GuildRoster();
		ChatSpamClean();
	end

	if player_summon_confirm then
		LazyPig_AutoSummon();
	end

	ScheduleButtonClick();
	ScheduleFunctionLaunch();
	ScheduleItemSplit();
	LazyPig_WatchSplit();
end

function ScheduleButtonClick(button, delay)
	local current_time = GetTime()
	if button and not ScheduleButton[button] then
		delay = delay or 0.75
		ScheduleButton[button] = current_time + delay
	else
		for blockindex,blockmatch in pairs(ScheduleButton) do
			if current_time < delayaction then
				ScheduleButton[blockindex] = nil
			elseif current_time >= blockmatch then
				blockindex:Click()
				passpopup = current_time + 0.75
				ScheduleButton[blockindex] = nil
			end
		end
	end
end

function ScheduleFunctionLaunch(func, delay)
	local current_time = GetTime()
	if func and not ScheduleFunction[func] then
		delay = delay or 0.75
		ScheduleFunction[func] = current_time + delay
	else
		for blockindex,blockmatch in pairs(ScheduleFunction) do
			if current_time >= blockmatch then
				blockindex()
				ScheduleFunction[blockindex] = nil
			end
		end
	end
end

local ErrorDismountAndForm = {
	[SPELL_FAILED_NOT_MOUNTED] = 1,                  -- "You are mounted"
	[ERR_ATTACK_MOUNTED] = 1,                        -- "Can't attack while mounted."
	[ERR_TAXIPLAYERALREADYMOUNTED] = 1,              -- "You are already mounted! Dismount first."
	[ERR_MOUNT_SHAPESHIFTED] = 1,                    -- "You can't mount while shapeshifted!"
	[SPELL_FAILED_NOT_SHAPESHIFT] = 1,               -- "You are in shapeshift form"
	[SPELL_FAILED_NO_ITEMS_WHILE_SHAPESHIFTED] = 1,  -- "Can't use items while shapeshifted"
	[SPELL_NOT_SHAPESHIFTED] = 1,                    -- "Can't do that while shapeshifted."
	[SPELL_NOT_SHAPESHIFTED_NOSPACE] = 1,            -- "Can't do that while shapeshifted."
	[ERR_TAXIPLAYERSHAPESHIFTED] = 1,                -- "You can't take a taxi while shapeshifted!"
	[ERR_CANT_INTERACT_SHAPESHIFTED] = 1,            -- "Can't speak while shapeshifted."
	[ERR_NO_ITEMS_WHILE_SHAPESHIFTED] = 1,           -- "Can't use items while shapeshifted."
	[ERR_NOT_WHILE_SHAPESHIFTED] = 1                 -- "You can't do that while shapeshifted."
}
local ErrorStanding = {
	[ERR_TAXINOTSTANDING] = 1,                       -- "You need to be standing to go anywhere."
	[ERR_LOOT_NOTSTANDING] = 1,                      -- "You need to be standing up to loot something!"
	[ERR_CANTATTACK_NOTSTANDING] = 1,                -- "You have to be standing to attack anything!"
	[SPELL_FAILED_NOT_STANDING] = 1                  -- "You must be standing to do that"
}

function LazyPig_OnEvent(event)
	if event == "ADDON_LOADED" and arg1 == "_LazyPig" then
		this:UnregisterEvent("ADDON_LOADED")
		local title = GetAddOnMetadata("_LazyPig", "Title")
		local version = GetAddOnMetadata("_LazyPig", "Version")
		DEFAULT_CHAT_FRAME:AddMessage(title.." v"..version.."|cffffffff".." loaded, type".."|cff00eeee".." /lp".."|cffffffff for options")

	elseif event == "PLAYER_LOGIN" then
		LazyPig_CreateOptionsFrame()
		LazyPig_CreateKeybindsFrame()

		LazyPig_CheckSalvation();
		LazyPig_CheckManaBuffs();
		Check_Bg_Status();
		LazyPig_AutoLeaveBG();
		LazyPig_AutoSummon();
		ScheduleFunctionLaunch(LazyPig_ZoneCheck, 6);
		LazyPig_MailtoCheck();

		if LPCONFIG.CAM then
			SetCVar("cameraDistanceMax",50)
		end
		if LPCONFIG.LOOT then
			UIPanelWindows["LootFrame"] = nil
		end
		QuestRecord["index"] = 0

	elseif (event == "UNIT_INVENTORY_CHANGED" and arg1 == "player") or event == "PLAYER_AURAS_CHANGED" or (event == "UPDATE_BONUS_ACTIONBAR" and LazyPig_PlayerClass("Druid")) then
		LazyPig_CheckSalvation()
		LazyPig_CheckManaBuffs()

	elseif event == "DUEL_REQUESTED" then
		duel_active = true
		if LPCONFIG.DUEL and not IsShiftKeyDown() then
			duel_active = nil
			CancelDuel()
			UIErrorsFrame:AddMessage(arg1.." - Duel Cancelled")
		end

	elseif event == "PLAYER_DEAD" then
		if LPCONFIG.RBG and LazyPig_BG() then
			RepopMe();
		end

	elseif event == "ZONE_CHANGED_NEW_AREA" or event == "PLAYER_UNGHOST" then
		if event == "ZONE_CHANGED_NEW_AREA" then
			tradestatus = nil
			mailstatus = nil
			auctionstatus = nil
			bankstatus = nil
			wsgefc = nil
		end

		ScheduleFunctionLaunch(LazyPig_ZoneCheck, 5)
		--DEFAULT_CHAT_FRAME:AddMessage(event);

	elseif event == "BANKFRAME_OPENED" then
		bankstatus = true
		tmp_splitval = 1

	elseif event == "BANKFRAME_CLOSED" then
		bankstatus = false
		LazyPig_EndSplit()

	elseif event == "AUCTION_HOUSE_SHOW" then
		auctionstatus = true
		auctionbrowse = nil
		tmp_splitval = 1

	elseif event == "AUCTION_HOUSE_CLOSED" then
		auctionstatus = false
		LazyPig_EndSplit()

	elseif event == "MAIL_SHOW" then
		mailstatus = true
		tmp_splitval = 1

	elseif event == "MAIL_CLOSED" then
		mailstatus = false
		LazyPig_EndSplit()

	elseif event == "MERCHANT_SHOW" then
		merchantstatus = true

	elseif event == "MERCHANT_CLOSED" then
		merchantstatus = false

	elseif event == "TRADE_SHOW" then
		tradestatus = true
		tmp_splitval = 1

	elseif event == "TRADE_CLOSED" then
		tradedelay = GetTime() + 1
		tradestatus = false
		LazyPig_EndSplit()

	elseif event == "START_LOOT_ROLL" then
		LazyPig_AutoRoll(arg1)

	elseif event == "UI_ERROR_MESSAGE" then
		if ErrorStanding[arg1] then
			SitOrStand()
		else
			if LPCONFIG.DISMOUNT and ErrorDismountAndForm[arg1] then
				UIErrorsFrame:Clear()
				LazyPig_Dismount()
				LazyPig_CancelShapeshiftBuff()
			end
			if LPCONFIG.AUTOSTANCE then
				LazyPig_AutoStance(arg1)
			end
		end
	elseif event == "UI_INFO_MESSAGE" then
		if arg1 == ERR_DUEL_CANCELLED then -- "Duel cancelled"
			duel_active = nil
		end
	elseif event == "CHAT_MSG_SYSTEM" then
		if arg1 == CLEARED_DND or arg1 == CLEARED_AFK then
			afk_active = false
			Check_Bg_Status()

		elseif string.find(arg1, string.sub(MARKED_DND, 1, string.len(MARKED_DND) -3)) then
			afk_active = false

		elseif string.find(arg1, string.sub(MARKED_AFK, 1, string.len(MARKED_AFK) -2)) then
			afk_active = true
			if LPCONFIG.EBG and not LazyPig_Raid() and not LazyPig_Dungeon() then
				UIErrorsFrame:AddMessage("Auto Join BG Inactive - AFK")
			end

		elseif LPCONFIG.AQUE and string.find(arg1 ,"Queued") and UnitIsPartyLeader("player") then
			if UnitInRaid("player") then
				SendChatMessage(arg1, "RAID");
			elseif GetNumPartyMembers() > 1 then
				SendChatMessage(arg1, "PARTY");
			end

		elseif string.find(arg1 ,"completed.") then
			LazyPig_FixQuest(arg1)
			QuestRecord["progress"] = nil

		elseif string.find(arg1 ,"Duel starting:") or string.find(arg1 ,"requested a duel") then
			duel_active = true
		elseif string.find(arg1 ,"in a duel") then
			duel_active = nil
		end

	elseif event == "QUEST_GREETING" then
		ActiveQuest = twipe(ActiveQuest)
		AvailableQuest = twipe(AvailableQuest)
		for i=1, GetNumActiveQuests() do
			table.insert(ActiveQuest, i, GetActiveTitle(i).." "..GetActiveLevel(i))
		end
		for i=1, GetNumAvailableQuests() do
			table.insert(AvailableQuest, i, GetAvailableTitle(i).." "..GetAvailableLevel(i))
		end

		LazyPig_ReplyQuest(event);

		--DEFAULT_CHAT_FRAME:AddMessage("active_: "..table.getn(ActiveQuest))
		--DEFAULT_CHAT_FRAME:AddMessage("available_: "..table.getn(AvailableQuest))

	elseif event == "GOSSIP_SHOW" then
		GossipOptions = twipe(GossipOptions)
		local dsc = nil
		local gossipnr = nil
		local gossipbreak = nil
		local processgossip = LPCONFIG.GOSSIP and not IsShiftKeyDown()

		dsc,GossipOptions[1],_,GossipOptions[2],_,GossipOptions[3],_,GossipOptions[4],_,GossipOptions[5] = GetGossipOptions()

		ActiveQuest = LazyPig_ProcessQuests(GetGossipActiveQuests())
		AvailableQuest = LazyPig_ProcessQuests(GetGossipAvailableQuests())

		if QuestRecord["qnpc"] ~= UnitName("npc") then
			QuestRecord["index"] = 0
			QuestRecord["qnpc"] = UnitName("npc")
		end

		if table.getn(AvailableQuest) ~= 0 or table.getn(ActiveQuest) ~= 0 then
			gossipbreak = true
		end

		--DEFAULT_CHAT_FRAME:AddMessage("gossip: "..table.getn(GossipOptions))
		--DEFAULT_CHAT_FRAME:AddMessage("active: "..table.getn(ActiveQuest))
		--DEFAULT_CHAT_FRAME:AddMessage("available: "..table.getn(AvailableQuest))

		for i=1, 5 do
			if not GossipOptions[i] then
				break
			end
			if GossipOptions[i] == "binder" then
				local bind = GetBindLocation();
				if not (bind == GetSubZoneText() or bind == GetZoneText() or bind == GetRealZoneText() or bind == GetMinimapZoneText()) then
					gossipbreak = true
				end
			elseif gossipnr then
				gossipbreak = true
			elseif GossipOptions[i] == "trainer" and dsc == "Reset my talents." then
				gossipbreak = false
			elseif ((GossipOptions[i] == "trainer" and processgossip)
					or (GossipOptions[i] == "vendor" and processgossip)
					or (GossipOptions[i] == "battlemaster" and (LPCONFIG.QBG or processgossip))
					or (GossipOptions[i] == "gossip" and processgossip)
					or (GossipOptions[i] == "banker" and string.find(dsc, "^I would like to check my deposit box.") and processgossip)
					or (GossipOptions[i] == "petition" and (IsAltKeyDown()or IsShiftKeyDown() or string.find(dsc, "Teleport me to the Molten Core")) and processgossip))
				then
				gossipnr = i
			elseif GossipOptions[i] == "taxi" and processgossip then
				gossipnr = i
				LazyPig_Dismount();
			end
		end

		if not gossipbreak and gossipnr then
			SelectGossipOption(gossipnr);
		else
			LazyPig_ReplyQuest(event);
		end

	elseif event == "QUEST_PROGRESS" or event == "QUEST_COMPLETE" then
		LazyPig_ReplyQuest(event);

	elseif event == "CHAT_MSG_BG_SYSTEM_ALLIANCE" or event == "CHAT_MSG_BG_SYSTEM_HORDE" then
		--DEFAULT_CHAT_FRAME:AddMessage(event.." - "..arg1);
		LazyPig_Track_EFC(arg1)

	elseif event == "UPDATE_BATTLEFIELD_STATUS" and not afk_active or event == "CHAT_MSG_BG_SYSTEM_NEUTRAL" and arg1 and string.find(arg1, "wins!") then
		bgstatus = GetTime()

	elseif event == "BATTLEFIELDS_SHOW" then
		LazyPig_QueueBG();

	elseif event == "CONFIRM_SUMMON" then
		LazyPig_AutoSummon();

	elseif event == "PARTY_INVITE_REQUEST" then
		local check1 = not LPCONFIG.DINV or LPCONFIG.DINV and not LazyPig_BG() and not LazyPig_Queue()
		local check2 = LPCONFIG.GINV and IsGuildMate(arg1) or LPCONFIG.FINV and IsFriend(arg1) or not IsGuildMate(arg1) and not IsFriend(arg1) and LPCONFIG.SINV
		if check1 and check2 then
			AcceptGroupInvite();
		end
	elseif event == "RESURRECT_REQUEST" and LPCONFIG.REZ then
		UIErrorsFrame:AddMessage(arg1.." - Resurrection")
		TargetByName(arg1, true)
		if GetCorpseRecoveryDelay() == 0 and (LazyPig_Raid() or LazyPig_Dungeon() or LazyPig_BG()) and UnitIsPlayer("target") and UnitIsVisible("target") and not UnitAffectingCombat("target") then
			AcceptResurrect()
			StaticPopup_Hide("RESURRECT_NO_TIMER");
			StaticPopup_Hide("RESURRECT_NO_SICKNESS");
			StaticPopup_Hide("RESURRECT");
		end
		TargetLastTarget();
	end
	--DEFAULT_CHAT_FRAME:AddMessage(event);
end

function LazyPig_StaticPopup_OnShow()
	if this.which == "QUEST_ACCEPT" and LazyPig_BG() and LPCONFIG.SBG then
		UIErrorsFrame:Clear();
		UIErrorsFrame:AddMessage("Quest Blocked Successfully");
		this:Hide();
	else
		Original_StaticPopup_OnShow();
	end
end

function LazyPig_MailtoCheck(msg)
	if MailTo_Option then -- to avoid conflicts with mailto addon
		local disable = LPCONFIG.RIGHT or LPCONFIG.SHIFT
		MailTo_Option.noshift = disable
		MailTo_Option.noauction = disable
		MailTo_Option.notrade = disable
		MailTo_Option.noclick = disable
		if msg then
			DEFAULT_CHAT_FRAME:AddMessage("_LazyPig: Warning Improved Right Click and Easy Split/Merge features may override MailTo addon functionality !")
		end
	end
end

function LazyPig_Text(txt)
	if txt then
		LazyPigText:SetTextColor(0, 1, 0)
		LazyPigText:SetText(txt)
		LazyPigText:Show()
	else
		LazyPigText:SetText()
		LazyPigText:Hide()
	end
end

--code taken from quickloot
local function LazyPig_ItemUnderCursor()
	if LPCONFIG.LOOT then
		local x, y = GetCursorPosition();
		local scale = LootFrame:GetEffectiveScale();
		x = x / scale;
		y = y / scale;
		LootFrame:ClearAllPoints();
		for index = 1, LOOTFRAME_NUMBUTTONS, 1 do
			local button = _G["LootButton"..index];
			if  button:IsVisible() then
				x = x - 42;
				y = y + 56 + (40 * index);
				LootFrame:SetPoint("TOPLEFT", "UIParent", "BOTTOMLEFT", x, y);
				return;
			end
		end
		if LootFrameDownButton:IsVisible() then
			x = x - 158;
			y = y + 223;
		else
			if GetNumLootItems() == 0  then
				HideUIPanel(LootFrame);
				return
			end
			x = x - 173;
			y = y + 25;
		end
		LootFrame:SetPoint("TOPLEFT", "UIParent", "BOTTOMLEFT", x, y);
	end
end

function LazyPig_LootFrame_OnEvent(event)
	OriginalLootFrame_OnEvent(event);
	if event == "LOOT_SLOT_CLEARED" then
		LazyPig_ItemUnderCursor();
	end
end

function LazyPig_LootFrame_Update()
	OriginalLootFrame_Update();
	LazyPig_ItemUnderCursor();
end

function IsFriend(name)
	for i = 1, GetNumFriends() do
		if GetFriendInfo(i) == name then
			return true
		end
	end
	return false
end

function IsGuildMate(name)
	if IsInGuild() then
		for i = 1, GetNumGuildMembers() do
			if string.lower(GetGuildRosterInfo(i)) == string.lower(name) then
				return true
			end
		end
	end
	return false
end

function AcceptGroupInvite()
	AcceptGroup()
	StaticPopup_Hide("PARTY_INVITE")
	PlaySoundFile("Sound\\Doodad\\BellTollNightElf.wav")
	UIErrorsFrame:AddMessage("Group Auto Accept")
end

function LazyPig_AutoSummon()
	if not LPCONFIG.SUMM then
		return
	end
	local keyenter = IsAltKeyDown() and IsControlKeyDown() and not tradestatus and not mailstatus and not auctionstatus and GetTime() > delayaction and GetTime() > (tradedelay + 0.5)
	local expireTime = GetSummonConfirmTimeLeft()
	if not player_summon_message and expireTime ~= 0 then
		player_summon_message = true
		player_summon_confirm = true
		DEFAULT_CHAT_FRAME:AddMessage("LazyPig: Auto Summon in "..math.floor(expireTime).."s", 1.0, 1.0, 0.0);

	elseif expireTime <= 3 or keyenter then
		player_summon_confirm = false
		player_summon_message = false

		for i=1,STATICPOPUP_NUMDIALOGS do
			local frame = _G["StaticPopup"..i]
			if frame.which == "CONFIRM_SUMMON" and frame:IsShown() then
				ConfirmSummon();
				delayaction = GetTime() + 0.75
				StaticPopup_Hide("CONFIRM_SUMMON");
			end
		end
	elseif expireTime == 0 then
		player_summon_confirm = false
		player_summon_message = false
	end
end

local bgStatus = {}
for i = 1, MAX_BATTLEFIELD_QUEUES do
    bgStatus[i] = { status = "", map = "", id = 0 }
end

function Check_Bg_Status()
	local player_bg_active = false
	local player_bg_request = false

	for i=1, MAX_BATTLEFIELD_QUEUES do
		local status, mapName, instanceID = GetBattlefieldStatus(i);
		for k in pairs(bgStatus[i]) do
			bgStatus[i][k] = nil
		end
		bgStatus[i]["status"] = status;
		bgStatus[i]["map"] = mapName;
		bgStatus[i]["id"] = instanceID;

		if status == "confirm" then
			player_bg_request = true
		elseif (status == "active") and not (mapName == "Eastern Kingdoms") and not (mapName == "Kalimdor") then
			player_bg_active = true
		end
	end

	player_bg_confirm = player_bg_request

	if player_bg_message and not player_bg_active and not player_bg_request then
		player_bg_message = false
	end

	if not player_bg_active and player_bg_request then
		local index = 1
		while bgStatus[index] do
			if bgStatus[index]["status"] == "confirm" then
				LazyPig_AutoJoinBG(index, bgStatus[index]["map"]);
			end
			index = index + 1
		end
	end
end

function LazyPig_QueueBG()
	if LPCONFIG.QBG then
		for i=1, MAX_BATTLEFIELD_QUEUES do
			local status = GetBattlefieldStatus(i);
			if IsShiftKeyDown() and (status == "queued" or status == "confirm") then
				AcceptBattlefieldPort(i,nil);
			end
		end
		if (GetNumPartyMembers() > 0 or GetNumRaidMembers() > 0) and IsPartyLeader() then
			JoinBattlefield(0,1);
		else
			JoinBattlefield(0);
		end
		ClearTarget();
		BattlefieldFrameCancelButton:Click()
	end
end

function LazyPig_AutoJoinBG(index, map_name)
	local keyenter = IsAltKeyDown() and IsControlKeyDown() and not tradestatus and not mailstatus and not auctionstatus and GetTime() > delayaction and GetTime() > (tradedelay + 0.5)
	if LPCONFIG.EBG or keyenter then
		local expireTime = GetBattlefieldPortExpiration(index)/1000
		expireTime = math.floor(expireTime);
		if not player_bg_message and expireTime > 3 and GetTime() > delayaction then
			player_bg_message = true
			DEFAULT_CHAT_FRAME:AddMessage("LazyPig: Auto Join ".. map_name.." in "..expireTime.."s", 1.0, 1.0, 0.0)

		elseif expireTime <= 3 or keyenter then
			AcceptBattlefieldPort(index, true);
			StaticPopup_Hide("CONFIRM_BATTLEFIELD_ENTRY")
			delayaction = GetTime() + 0.75
			if player_bg_message then
				player_bg_message = false
			end
		end
	end
end

function LazyPig_AutoLeaveBG()
	local keyenter = IsAltKeyDown() and IsControlKeyDown()
	if LPCONFIG.LBG or keyenter then
		local bg_winner = GetBattlefieldWinner()
		local winner_name = "Alliance"
		if bg_winner ~= nil then
			if bg_winner == 0 then winner_name = "Horde" end
			UIErrorsFrame:Clear();
			UIErrorsFrame:AddMessage(winner_name.." Wins");
			LeaveBattlefield();
		end
	end
end

function LazyPig_BagReturn(find)
	local link = nil
	local bagslots = nil
	for bag=0,NUM_BAG_FRAMES do
		bagslots = GetContainerNumSlots(bag)
		if bagslots and bagslots > 0 then
			for slot=1,bagslots do
				link = GetContainerItemLink(bag, slot)
				if not find and not link or find and link and string.find(link, find) then
					return bag, slot
				end
			end
		end
	end
	return nil
end

local function RollToString(roll)
	local txt = ""
	if roll == 1 then
		txt = string.upper(NEED)
	elseif roll == 2 then
		txt = string.upper(GREED)
	elseif roll == 0 then
		txt = string.upper(PASS)
	end
	return txt
end

local ZGloot = {
	[19698] = "Zulian Coin",
	[19699] = "Razzashi Coin",
	[19700] = "Hakkari Coin",
	[19701] = "Gurubashi Coin",
	[19702] = "Vilebranch Coin",
	[19703] = "Witherbark Coin",
	[19704] = "Sandfury Coin",
	[19705] = "Skullsplitter Coin",
	[19706] = "Bloodscalp Coin",
	[19707] = "Red Hakkari Bijou",
	[19708] = "Blue Hakkari Bijou",
	[19709] = "Yellow Hakkari Bijou",
	[19710] = "Orange Hakkari Bijou",
	[19711] = "Green Hakkari Bijou",
	[19712] = "Purple Hakkari Bijou",
	[19713] = "Bronze Hakkari Bijou",
	[19714] = "Silver Hakkari Bijou",
	[19715] = "Gold Hakkari Bijou",
}

local MCloot = {
	[11382] = "Blood of the Mountain",
	[17010] = "Fiery Core",
	[17011] = "Lava Core",
}

local AQloot = {
	[20858] = "Stone Scarab",
	[20859] = "Gold Scarab",
	[20860] = "Silver Scarab",
	[20861] = "Bronze Scarab",
	[20862] = "Crystal Scarab",
	[20863] = "Clay Scarab",
	[20864] = "Bone Scarab",
	[20865] = "Ivory Scarab",
	[20866] = "Azure Idol",
	[20867] = "Onyx Idol",
	[20868] = "Lambent Idol",
	[20869] = "Amber Idol",
	[20870] = "Jasper Idol",
	[20871] = "Obsidian Idol",
	[20872] = "Vermillion Idol",
	[20873] = "Alabaster Idol",
	[20874] = "Idol of the Sun",
	[20875] = "Idol of Night",
	[20876] = "Idol of Death",
	[20877] = "Idol of the Sage",
	[20878] = "Idol of Rebirth",
	[20879] = "Idol of Life",
	[20881] = "Idol of Strife",
	[20882] = "Idol of War",
}

local BMloot = {
	[50203] = "Corrupted Sand",
}

local ESloot = {
	[20381] = "Dreamscale",
	[61197] = "Fading Dream Fragment",
	[61198] = "Small Dream Shard",
}

local NaxxLoot = {
	[22373] = "Wartorn Leather Scrap",
	[22374] = "Wartorn Chain Scrap",
	[22375] = "Wartorn Plate Scrap",
	[22376] = "Wartorn Cloth Scrap",
}

function LazyPig_AutoRoll(id)
	local roll = nil
	local _, _, _, quality = GetLootRollItemInfo(id)
	local link = GetLootRollItemLink(id)
	local _, _, itemID = strfind(link or "", "item:(%d+)")
	itemID = tonumber(itemID)
	
	if not itemID then
		return
	end
	
	if LPCONFIG.ZG and ZGloot[itemID] then
		roll = LPCONFIG.ZG
		RollOnLoot(id, LPCONFIG.ZG)
	end

	if LPCONFIG.MC and MCloot[itemID] then
		roll = LPCONFIG.MC
		RollOnLoot(id, LPCONFIG.MC)
	end

	if LPCONFIG.AQ and AQloot[itemID] then
		roll = LPCONFIG.AQ
		RollOnLoot(id, LPCONFIG.AQ)
	end

	if LPCONFIG.SAND and BMloot[itemID] then
		roll = LPCONFIG.SAND
		RollOnLoot(id, LPCONFIG.SAND)
	end

	-- Hard coded auto need for Necrotic Runes
	if itemID == 22484 then
		roll = 1
		RollOnLoot(id, 1)
	end
	-- Need on everything in Alterac Valley
	if LazyPig_BG() then
		roll = 1
		RollOnLoot(id, 1)
	end

	if LPCONFIG.ES_SHARDS and ESloot[itemID] then
		roll = LPCONFIG.ES_SHARDS
		RollOnLoot(id, LPCONFIG.ES_SHARDS)
	end

	if LPCONFIG.NAXX and NaxxLoot[itemID] then
		roll = LPCONFIG.NAXX
		RollOnLoot(id, LPCONFIG.NAXX)
	end

	if LPCONFIG.ROLLMSG and type(roll) == "number" then
		local _, _, _, hex = GetItemQualityColor(quality)
		DEFAULT_CHAT_FRAME:AddMessage("LazyPig: Auto "..hex..RollToString(roll).." "..GetLootRollItemLink(id))
	end

	-- Auto accept BoP for things that are auto rolled. Like Corrupted Sand and Necrotic Runes
	for i=1,STATICPOPUP_NUMDIALOGS do
		local frame = _G["StaticPopup"..i]
		if frame:IsShown() and frame.which == "CONFIRM_LOOT_ROLL" and frame.data == id and frame.data2 == roll then
			_G["StaticPopup"..i.."Button1"]:Click()
		end
	end
end

function LazyPig_GreenRoll()
	if not LPCONFIG.GREEN then
		return
	end
	local pass = nil
	for i=1, NUM_GROUP_LOOT_FRAMES do
		local frame = _G["GroupLootFrame"..i];
		if frame:IsVisible() then
			local id = frame.rollID
			local _, name, _, quality = GetLootRollItemInfo(id);
			if quality == 2 then
				RollOnLoot(id, LPCONFIG.GREEN);
				local _, _, _, hex = GetItemQualityColor(quality)
				greenrolltime = GetTime() + 1
				if LPCONFIG.ROLLMSG then
					DEFAULT_CHAT_FRAME:AddMessage("LazyPig: Auto "..hex..RollToString(LPCONFIG.GREEN).." "..GetLootRollItemLink(id))
				end
				pass = true
			end
		end
	end
	return pass
end

local COLOR_COPPER = "|cffeda55f"
local COLOR_SILVER = "|cffc7c7cf"
local COLOR_GOLD = "|cffffd700"

local function MoneyToString(money)
	if not money then
		return ""
	end
	local gold = floor(abs(money / 10000))
	local silver = floor(abs(mod(money / 100, 100)))
	local copper = floor(abs(mod(money, 100)))
	return COLOR_GOLD..gold.."g|r "..COLOR_SILVER..silver.."s|r "..COLOR_COPPER..copper.."c|r"
end

function LazyPig_GreySellRepair()
	if C_MerchantFrame then
		if C_MerchantFrame.GetNumJunkItems() > 0 then C_MerchantFrame.SellAllJunkItems() end
	else
		local i = 0
		for bag = 0, NUM_BAG_FRAMES do
			for slot = 1, GetContainerNumSlots(bag) do
				local link = GetContainerItemLink(bag, slot)
				local _, _, locked = GetContainerItemInfo(bag, slot)
				local _, _, id = string.find(link or "", "item:(%d+)")
				id = tonumber(id)
				local _, _, quality = GetItemInfo(id or 0)
				if quality and quality == 0 and not locked then
					UseContainerItem(bag, slot)
					i = i + 1
					if i > 4 then
						bag = NUM_BAG_FRAMES
						break
					end
				end
			end
		end
	end
	if not CanMerchantRepair() then return end
	local rcost = GetRepairAllCost()
	if rcost == 0 then return end
	if rcost > GetMoney() then
		DEFAULT_CHAT_FRAME:AddMessage("LazyPig: Not enough money to repair")
		return
	end
	RepairAllItems()
	DEFAULT_CHAT_FRAME:AddMessage("LazyPig: Repaired all items for "..MoneyToString(rcost))
end

function LazyPig_ProcessQuests(...)
	local quest = {}
	for i = 1, table.getn(arg), 2 do
		local count, title, level = i, arg[i], arg[i+1]
		if count > 1 then count = (count+1)/2 end
		quest[count] = title.." "..level
	end
	return quest
end

function LazyPig_SelectGossipActiveQuest(index, norecord)
	if not ActiveQuest[index] then
		--DEFAULT_CHAT_FRAME:AddMessage("LazyPig_SelectGossipActiveQuest Error");
	elseif not norecord then
		LazyPig_RecordQuest(ActiveQuest[index])
	end
	Original_SelectGossipActiveQuest(index);
end

function LazyPig_SelectGossipAvailableQuest(index, norecord)
	if not AvailableQuest[index] then
		--DEFAULT_CHAT_FRAME:AddMessage("LazyPig_SelectGossipAvailableQuest Error");
	elseif not norecord then
		LazyPig_RecordQuest(AvailableQuest[index])
	end
	Original_SelectGossipAvailableQuest(index);
end

function LazyPig_SelectActiveQuest(index, norecord)
	if not ActiveQuest[index] then
		--DEFAULT_CHAT_FRAME:AddMessage("LazyPig_SelectActiveQuest Error");
	elseif not norecord then
		LazyPig_RecordQuest(ActiveQuest[index])
	end
	Original_SelectActiveQuest(index);
end

function LazyPig_SelectAvailableQuest(index, norecord)
	if not AvailableQuest[index] then
		--DEFAULT_CHAT_FRAME:AddMessage("LazyPig_SelectAvailableQuest Error");
	elseif not norecord then
		LazyPig_RecordQuest(AvailableQuest[index])
	end
	Original_SelectAvailableQuest(index);
end

function LazyPig_FixQuest(quest, annouce)
	if QuestHaste then
		return
	end
	if not QuestRecord["details"] then
		annouce = true
	end
	if UnitLevel("player") == 60 then
		if string.find(quest, "Fight for Warsong Gulch") then
			QuestRecord["details"] = "Fight for Warsong Gulch 60"
		elseif string.find(quest, "Battle of Warsong Gulch") then
			QuestRecord["details"] = "Battle of Warsong Gulch 60"
		elseif string.find(quest, "Claiming Arathi Basin") then
			QuestRecord["details"] = "Claiming Arathi Basin 60"
		elseif string.find(quest, "Conquering Arathi Basin") then
			QuestRecord["details"] = "Conquering Arathi Basin 60"
		end
	end
	if QuestRecord["details"] and annouce then
		UIErrorsFrame:Clear();
		UIErrorsFrame:AddMessage("Recording: "..QuestRecord["details"])
	end
end

function LazyPig_RecordQuest(qdetails)
	if QuestHaste then
		return
	end
	if IsAltKeyDown() and qdetails then
		if QuestRecord["details"] ~= qdetails then
			QuestRecord["details"] = qdetails
		end
		LazyPig_FixQuest(QuestRecord["details"], true)
	elseif not IsAltKeyDown() and QuestRecord["details"] then
		QuestRecord["details"] = nil
		QuestRecord.itemChoice = nil
	end
	QuestRecord["progress"] = true
end

function LazyPig_QuestRewardItem_OnClick()
	Original_QuestRewardItem_OnClick()
	if QuestRecord.details and this.type == "choice" then
		QuestRewardItemHighlight:SetPoint("TOPLEFT", this, "TOPLEFT", -8, 7);
		QuestRewardItemHighlight:Show();
		QuestFrameRewardPanel.itemChoice = this:GetID();
		QuestRecord.itemChoice = this:GetID();
	end
end

function LazyPig_ReplyQuest(event)
	if QuestHaste or not IsAltKeyDown() then
		return
	end

	if QuestRecord["details"] then
		UIErrorsFrame:Clear();
		UIErrorsFrame:AddMessage("Replaying: "..QuestRecord["details"])
	end

	if event == "GOSSIP_SHOW" then
		if QuestRecord["details"] then
			for blockindex,blockmatch in pairs(ActiveQuest) do
				if blockmatch == QuestRecord["details"] then
					Original_SelectGossipActiveQuest(blockindex)
					return
				end
			end
			for blockindex,blockmatch in pairs(AvailableQuest) do
				if blockmatch == QuestRecord["details"] then
					Original_SelectGossipAvailableQuest(blockindex)
					return
				end
			end
		elseif table.getn(ActiveQuest) == 0 and table.getn(AvailableQuest) == 1 or IsAltKeyDown() and table.getn(AvailableQuest) > 0 then
			LazyPig_SelectGossipAvailableQuest(1, true)
		elseif table.getn(ActiveQuest) == 1 and table.getn(AvailableQuest) == 0 or IsAltKeyDown() and table.getn(ActiveQuest) > 0 then
			local nr = table.getn(ActiveQuest)
			if QuestRecord["progress"] and (nr - QuestRecord["index"]) > 0 then
				--DEFAULT_CHAT_FRAME:AddMessage("++quest dec nr - "..nr.." index - "..QuestRecord["index"])
				QuestRecord["index"] = QuestRecord["index"] + 1
				nr = nr - QuestRecord["index"]
			end
			LazyPig_SelectGossipActiveQuest(nr, true)
		end
	elseif event == "QUEST_GREETING" then
		if QuestRecord["details"] then
			for blockindex,blockmatch in pairs(ActiveQuest) do
				if blockmatch == QuestRecord["details"] then
					Original_SelectActiveQuest(blockindex)
					return
				end
			end
			for blockindex,blockmatch in pairs(AvailableQuest) do
				if blockmatch == QuestRecord["details"] then
					Original_SelectAvailableQuest(blockindex)
					return
				end
			end
		elseif table.getn(ActiveQuest) == 0 and table.getn(AvailableQuest) == 1 or IsAltKeyDown() and table.getn(AvailableQuest) > 0 then
			LazyPig_SelectAvailableQuest(1, true)
		elseif table.getn(ActiveQuest) == 1 and table.getn(AvailableQuest) == 0 or IsAltKeyDown() and table.getn(ActiveQuest) > 0 then
			local nr = table.getn(ActiveQuest)
			if QuestRecord["progress"] and (nr - QuestRecord["index"]) > 0 then
				--DEFAULT_CHAT_FRAME:AddMessage("--quest dec nr - "..nr.." index - "..QuestRecord["index"])
				QuestRecord["index"] = QuestRecord["index"] + 1
				nr = nr - QuestRecord["index"]
			end
			LazyPig_SelectActiveQuest(nr, true)
		end

	elseif event == "QUEST_PROGRESS" then
		CompleteQuest()
	elseif event == "QUEST_COMPLETE" then
		if GetNumQuestChoices() == 0 then
			GetQuestReward(0)
		elseif GetNumQuestChoices() > 0 and QuestRecord.itemChoice then
			GetQuestReward(QuestRecord.itemChoice)
		end
	end
end

-- taken from ShaguTweaks
-- https://github.com/shagu/ShaguTweaks/blob/master/mods/auto-dismount.lua
local dismountStrings = {
	"^Increases speed by (.+)%%", -- enUS
	"^Erhöht Tempo um (.+)%%", -- deDE
	"^Aumenta la velocidad en un (.+)%%", -- esES
	"^Augmente la vitesse de (.+)%%", -- frFR
	"^Скорость увеличена на (.+)%%", -- ruRU
	"^이동 속도 (.+)%%만큼 증가", -- koKR
	"^速度提高(.+)%%", -- zhCN
	-- turtle-wow
	"speed based on", "Slow and steady...", "Riding",
	"Lento y constante...", "Aumenta la velocidad según tu habilidad de Montar.",
	"根据您的骑行技能提高速度。", "根据骑术技能提高速度。", "又慢又稳......",
}

function LazyPig_Dismount()
	if Dismount then return Dismount() end
	local buff = 0
	while GetPlayerBuff(buff) >= 0 do
		LazyPig_Buff_Tooltip:SetOwner(WorldFrame, "ANCHOR_NONE")
		LazyPig_Buff_Tooltip:SetPlayerBuff(GetPlayerBuff(buff))
		local desc = LazyPig_Buff_TooltipTextLeft2:GetText()
		if desc then
			for _, str in pairs(dismountStrings) do
				if string.find(desc, str) then return CancelPlayerBuff(buff) end
			end
		end
		buff = buff + 1
	end
end

local stancesPattern = string.gsub(SPELL_FAILED_ONLY_SHAPESHIFT, "%%s", "(.+)")

function LazyPig_AutoStance(msg)
	local _, _, stances = string.find(msg, stancesPattern)
	if not stances then return end
	for stance in string.gfind(stances, "[^,]+") do
		CastSpellByName((string.gsub(stance, "^%s*(.-)%s*$", "%1")))
	end
end

function LazyPig_DropWSGFlag_NoggBuff()
	local counter = 0
	local tooltipfind1 = "Warsong Flag"
	local tooltipfind2 = "You feel light"
	local tooltipfind3 = "Slow Fall"

	while GetPlayerBuff(counter) >= 0 do
		local index, untilCancelled = GetPlayerBuff(counter)
		LazyPig_Buff_Tooltip:SetOwner(WorldFrame, "ANCHOR_NONE")
		LazyPig_Buff_Tooltip:SetPlayerBuff(index)
		local desc = LazyPig_Buff_TooltipTextLeft1:GetText()
		if string.find(desc, tooltipfind1) or string.find(desc, tooltipfind3) then
			CancelPlayerBuff(counter)
		end
		desc = LazyPig_Buff_TooltipTextLeft2:GetText()
		if string.find(desc, tooltipfind2) then
			CancelPlayerBuff(counter)
		end
		counter = counter + 1
	end
end

function LazyPig_ItemIsTradeable(bag, item)
	LazyPig_Buff_Tooltip:SetOwner(WorldFrame, "ANCHOR_NONE")
	LazyPig_Buff_Tooltip:SetBagItem(bag, item)
	for i = 1, LazyPig_Buff_Tooltip:NumLines(), 1 do
		local text = _G["LazyPig_Buff_TooltipTextLeft" .. i]:GetText()
		if text == ITEM_SOULBOUND or text == ITEM_BIND_QUEST or text == ITEM_CONJURED then
			return false
		end
	end
	return true
end

function LazyPig_Raid()
	local inInstance, instanceType = IsInInstance()
	return inInstance and instanceType == "raid"
end

function LazyPig_Dungeon()
	local inInstance, instanceType = IsInInstance()
	return inInstance and instanceType == "party"
end

function LazyPig_BG()
	local inInstance, instanceType = IsInInstance()
	return inInstance and instanceType == "pvp"
end

function LazyPig_Queue()
	for i = 1, MAX_BATTLEFIELD_QUEUES do
		local status, mapName, instanceID = GetBattlefieldStatus(i)
		if status == "confirm" or status == "active" then
			return true
		end
	end
	return false
end

function LazyPig_EndSplit()
	timer_split = nil
	tmp_splitval = 1
	LazyPig_Text()
end

function LazyPig_DecodeItemLink(link)
	if not link then return nil end
	local _, _, itemString, name = string.find(link, "|H(item:.*)|h%[(.*)%]|h")
	return name, itemString
end

function LazyPig_WatchSplit(enable)
	local returnval = timer_split
	if LPCONFIG.SHIFTSPLIT then
		local time = GetTime()
		local txt_show = enable
		local ctrl = IsControlKeyDown()
		local alt = IsAltKeyDown()
		local duration = 9

		if enable then
			timer_split = time + duration
		elseif timer_split then
			local boost = -0.006*tmp_splitval + 1

			if auctionstatus then
				if AuctionFrameBrowse:IsVisible() then
					if not auctionbrowse then
						auctionbrowse = true
						LazyPig_EndSplit()
						return
					end
				else
					auctionbrowse = nil
				end
			end

			if time > timer_split or auctionstatus and AuctionFrameAuctions and not (AuctionFrameAuctions:IsVisible() or AuctionFrameBrowse:IsVisible()) or mailstatus and SendMailFrame and not SendMailFrame:IsVisible() and (not CT_MailFrame or CT_MailFrame and not CT_MailFrame:IsVisible()) and (not GMailFrame or GMailFrame and not GMailFrame:IsVisible()) then
				LazyPig_EndSplit()
			elseif (ctrl or alt) and time > last_click then
				local forcepass = (timer_split - duration + 0.6) > time
				if ctrl and alt then
					timer_split = time + duration - 1
					return
				elseif alt and ((time + 0.1) > alttime or forcepass) and tmp_splitval < 100 then
					alttime = time + 0.125
					tmp_splitval = tmp_splitval + 1
					timer_split = time + duration
					last_click = 0.109*boost + time
					txt_show = true
				elseif ctrl and ((time + 0.1) > ctrltime or forcepass) and tmp_splitval > 1 then
					ctrltime = time + 0.125
					tmp_splitval = tmp_splitval - 1
					timer_split = time + duration
					last_click = 0.109*boost + time
					txt_show = true
				end
			end
		elseif auctionstatus and AuctionFrameAuctions and AuctionFrameAuctions:IsVisible() or mailstatus and SendMailFrame and SendMailFrame:IsVisible() or tradestatus or bankstatus or CT_MailFrame and CT_MailFrame:IsVisible() or GMailFrame and GMailFrame:IsVisible() then
			timer_split = time + duration - 1
			txt_show = true
		end
		if txt_show then LazyPig_Text("- Ctrl  "..tmp_splitval.."  Alt +") end
	end
	return returnval
end

local ItemArray = {}
function LazyPig_UseContainerItem(ParentID,ItemID)
	if LPCONFIG.SHIFTSPLIT and not CursorHasItem() and not merchantstatus and IsShiftKeyDown() and not IsAltKeyDown() then
		if(GetTime() - last_click) < 0.3 then return end
		local _, itemCount, locked = GetContainerItemInfo(ParentID, ItemID)
		if locked or not itemCount then return end
		if not LazyPig_WatchSplit(true) then return end
		last_click = GetTime()

		ItemArray = twipe(ItemArray)
		local name, id = LazyPig_DecodeItemLink(GetContainerItemLink(ParentID,ItemID))
		local _, _, _, _, _, _, maxstack = GetItemInfo(id)
		local out_slpit = tmp_splitval

		if out_slpit > maxstack then
			out_slpit = maxstack
		end

		local dcount = out_slpit
		local dbag = nil
		local dslot = nil
		local cursoritem = nil

		if itemCount < out_slpit then
			dbag, dslot = ParentID, ItemID
			dcount = out_slpit - itemCount
			cursoritem = true
		end

		if name then
			for b=0, NUM_BAG_FRAMES do
				local bagslots = GetContainerNumSlots(b)
				if bagslots and bagslots > 0 then
					for s=1, bagslots do
						local link = GetContainerItemLink(b, s)
						local n, c, l = GetContainerItemInfo(b, s)
						if not cursoritem or cursoritem and not (b == ParentID and s == ItemID) then
							if not link and not dbag and not dslot then
								dbag, dslot = b, s
								--DEFAULT_CHAT_FRAME:AddMessage(b.." "..s.." - scan mode1")
							elseif n then
								if n == name then
								--if (string.find(n, name) or n == name) then
									if not l then
										if not (itemCount < out_slpit) and not dbag and not dslot and c < out_slpit then
											dbag, dslot = b, s
											dcount = out_slpit - c
											--DEFAULT_CHAT_FRAME:AddMessage("b.." "..s.." count - "..c.." - "..scan mode2)
										elseif c ~= out_slpit or cursoritem then
											ItemArray[b.."_"..s] = c
										end
									end
								end
							end
						end
					end
				end
			end

			if not dbag or not dslot or CursorHasItem() then return end

			local escape = 0
			while dcount > 0 do
				local sbag = nil
				local sslot = nil
				local score = nil
				local number = nil
				local index = nil

				for blockindex,blockmatch in pairs(ItemArray) do
					local x = nil
					local y = nil
					x = string.gsub(blockindex,"_(.+)","")
					x = tonumber(x)
					y = string.gsub(blockindex,"(.+)_","")
					y = tonumber(y)

					if not number or number > blockmatch or number == blockmatch and (x*10 + y) > score then
						sbag = x
						sslot = y
						score = 10*sbag + sslot
						number = blockmatch
						index = blockindex
					end
				end

				if sbag and sslot then
					local splitval = nil
					if (number - dcount) >= out_slpit then
						splitval = dcount
					elseif number > out_slpit then
						splitval = number - out_slpit
					elseif number < dcount then
						splitval = number
					else
						splitval = dcount
					end

					dcount = dcount - splitval
					ScheduleItemSplit(sbag, sslot, dbag, dslot, splitval)
					ItemArray[index] = nil
					--DEFAULT_CHAT_FRAME:AddMessage("Dest "..dbag.." - "..dslot.." From "..sbag.." - "..sslot.." - Count "..splitval)
				end

				if escape > 99 then
					--DEFAULT_CHAT_FRAME:AddMessage("LazPig_Split: Loop stop")
					return
				else
					escape = escape + 1
				end
			end
		end
		return

	elseif LPCONFIG.RIGHT and tradestatus and not IsShiftKeyDown() and not IsAltKeyDown() and LazyPig_ItemIsTradeable(ParentID,ItemID) then
		PickupContainerItem(ParentID,ItemID)
		local slot = TradeFrame_GetAvailableSlot()
		if slot then ClickTradeButton(slot) end
		if CursorHasItem() then
			ClearCursor()
		end
		return

	elseif LPCONFIG.RIGHT and GMailFrame and GMailFrame:IsVisible() and not CursorHasItem() then
		local bag, item = ParentID,ItemID
		for i = 1, GMAIL_NUMITEMBUTTONS, 1 do
			if  not _G["GMailButton" .. i].item  then

				if  GMail:ItemIsMailable(bag, item)  then
					GMail:Print("GMail: Cannot attach item.", 1, 0.5, 0)
					return
				end
				PickupContainerItem(bag, item)
				--GMail.hooks["PickupContainerItem"].orig(bag, item)
				GMail:MailButton_OnClick(_G["GMailButton" .. i])
				GMail:UpdateItemButtons()
				return
			end
		end

	elseif LPCONFIG.RIGHT and CT_MailFrame and CT_MailFrame:IsVisible() and not IsShiftKeyDown() and not IsAltKeyDown() then
		local bag, item = ParentID,ItemID
		if  ( CT_Mail_GetItemFrame(bag, item) or ( CT_Mail_addItem and CT_Mail_addItem[1] == bag and CT_Mail_addItem[2] == item ) ) and not special  then
			return;
		end
		if  not CursorHasItem()  then
			CT_MailFrame.bag = bag;
			CT_MailFrame.item = item;
		end
		if  CT_MailFrame:IsVisible() and not CursorHasItem()  then
			for i = 1, CT_MAIL_NUMITEMBUTTONS, 1 do
				if  not _G["CT_MailButton" .. i].item then

					local canMail = CT_Mail_ItemIsMailable(bag, item);
					if  canMail  then
						DEFAULT_CHAT_FRAME:AddMessage("<CTMod> Cannot attach item, item is " .. canMail, 1, 0.5, 0);
						return;
					end

					CT_oldPickupContainerItem(bag, item);
					CT_MailButton_OnClick(_G["CT_MailButton" .. i]);
					CT_Mail_UpdateItemButtons();
					return;
				end
			end
		end

	elseif LPCONFIG.RIGHT and mailstatus and not IsShiftKeyDown() and not IsAltKeyDown() then
		if not LazyPig_ItemIsTradeable(ParentID,ItemID) then
			DEFAULT_CHAT_FRAME:AddMessage("LazyPig: Cannot attach item", 1, 0.5, 0);
			return
		end

		if InboxFrame and InboxFrame:IsVisible() then
			MailFrameTab_OnClick(2);
			return
		end
		if SendMailFrame and SendMailFrame:IsVisible() then
			PickupContainerItem(ParentID,ItemID)
			ClickSendMailItemButton()
			if CursorHasItem() then
				ClearCursor()
			end
			return
		end

	elseif LPCONFIG.RIGHT and auctionstatus and not IsShiftKeyDown() and not IsAltKeyDown() then
		if not LazyPig_ItemIsTradeable(ParentID,ItemID) then
			DEFAULT_CHAT_FRAME:AddMessage("LazyPig: Cannot sell item", 1, 0.5, 0);
			return
		end
		if not AuctionFrameAuctions:IsVisible() then
			AuctionFrameTab3:Click()
			return
		end
		PickupContainerItem(ParentID,ItemID)
		ClickAuctionSellItemButton()
		if CursorHasItem() then
			ClearCursor()
		end
		return
	end
	OriginalUseContainerItem(ParentID,ItemID)
end

function ScheduleItemSplit(sbag, sslot, dbag, dslot, count)
	if sbag and sslot and dbag and dslot and count then

		local number = nil

		for blockindex,blockmatch in pairs(ScheduleSplitCount) do
			if not number or number < blockindex then
				number = blockindex
			end
		end

		if not number then
			number = 1
		else
			number = number + 1
		end

		--DEFAULT_CHAT_FRAME:AddMessage("Task Count - "..number)

		ScheduleSplitCount[number] = true
		ScheduleSplit.sbag[number] = sbag
		ScheduleSplit.sslot[number] = sslot
		ScheduleSplit.dbag[number] = dbag
		ScheduleSplit.dslot[number] = dslot
		ScheduleSplit.count[number] = count

		ScheduleSplit.active = true

	elseif ScheduleSplit.active then

		local number = nil
		for blockindex,blockmatch in pairs(ScheduleSplitCount) do
			if not number or number > blockindex then
				number = blockindex
			end
		end

		if number then
			last_click = GetTime()
			local _, _, lock = GetContainerItemInfo(ScheduleSplit.dbag[number], ScheduleSplit.dslot[number])
			if not lock then

				--DEFAULT_CHAT_FRAME:AddMessage("Dest "..ScheduleSplit.dbag[number].." - "..ScheduleSplit.dslot[number].." From "..ScheduleSplit.sbag[number].." - "..ScheduleSplit.sslot[number].." - Count "..ScheduleSplit.count[number])

				SplitContainerItem(ScheduleSplit.sbag[number], ScheduleSplit.sslot[number], ScheduleSplit.count[number])
				PickupContainerItem(ScheduleSplit.dbag[number], ScheduleSplit.dslot[number])
				ScheduleSplitCount[number] = nil
			end
		else
			ScheduleSplit.active = nil
		end
	end
end

function LazyPig_RollLootOpen()
	for i=1,STATICPOPUP_NUMDIALOGS do
		local frame = _G["StaticPopup"..i]
		if frame:IsShown() and frame.which == "CONFIRM_LOOT_ROLL" then
			--DEFAULT_CHAT_FRAME:AddMessage("LazyPig_RollLootOpen - TRUE")
			return true
		end
	end
	return nil
end

function LazyPig_BindLootOpen()
	for i=1,STATICPOPUP_NUMDIALOGS do
		local frame = _G["StaticPopup"..i]
		if frame:IsShown() and frame.which == "LOOT_BIND" then
			--DEFAULT_CHAT_FRAME:AddMessage("LazyPig_BindLootOpen - TRUE")
			return true
		end
	end
	return nil
end

local process = function(ChatFrame, name)
    for index, value in pairs(ChatFrame.channelList) do
        if strupper(name) == strupper(value) then
            return true
        end
    end
    return nil
end

function LazyPig_ZoneCheck()
	local leavechat = LPCONFIG.WORLDRAID and LazyPig_Raid() or LPCONFIG.WORLDDUNGEON and LazyPig_Dungeon() or LPCONFIG.WORLDBG and LazyPig_BG() or LPCONFIG.WORLDUNCHECK
	for i = 1, NUM_CHAT_WINDOWS do
		local chatFrame = _G["ChatFrame"..i]
		if chatFrame:IsVisible() and not UnitIsDeadOrGhost("player") then
			local id, name = GetChannelName("world")
			if id > 0 then
				if leavechat then
					if process(chatFrame, name) then
						ChatFrame_RemoveChannel(chatFrame, name)
						channelstatus = true
						UIErrorsFrame:Clear();
						UIErrorsFrame:AddMessage("Leaving World")
					end
					return
				end
			end
			if (LPCONFIG.WORLDRAID or LPCONFIG.WORLDDUNGEON or LPCONFIG.WORLDBG) and not leavechat then
				local framename = chatFrame:GetName()
				if id == 0 then
					UIErrorsFrame:Clear();
					UIErrorsFrame:AddMessage("Joining World");
					JoinChannelByName("world", nil, chatFrame:GetID());
				else
					if (not process(chatFrame, name) or channelstatus) and framename == "ChatFrame1" then
						ChatFrame_AddChannel(chatFrame, name);
						UIErrorsFrame:Clear();
						UIErrorsFrame:AddMessage("Joining World");
						channelstatus = false
					end
				end
			end
		end
	end
end

function LazyPig_PlayerClass(class, unit)
	if class then
		unit = unit or "player"
		local _, c = UnitClass(unit)
		return c == string.upper(class)
	end
	return false
end

function LazyPig_IsBearForm()
	if GetShapeshiftFormID then
		local id = GetShapeshiftFormID()
		return id == 5 or id == 8
	end
	for i = 1 , GetNumShapeshiftForms() do
		local _, name, isActive = GetShapeshiftFormInfo(i)
		if isActive and LazyPig_PlayerClass("Druid") and (name == "Bear Form" or name == "Dire Bear Form") then
			return true
		end
	end
	return false
end

function LazyPig_IsShieldEquipped()
	local name, itemString = LazyPig_DecodeItemLink(GetInventoryItemLink("player", 17))
	if itemString then
		local _, _, _, _, _, _, _, invType = GetItemInfo(itemString)
		return invType == "INVTYPE_SHIELD"
	end
	return false
end

function LazyPig_CancelShapeshiftBuff()
	if CancelShapeshiftForm then return CancelShapeshiftForm() end
	for i = 1, GetNumShapeshiftForms() do
		local _, _, isActive = GetShapeshiftFormInfo(i)
		if isActive and LazyPig_PlayerClass("Druid") then
			CastShapeshiftForm(i)
			return
		end
	end
end

function LazyPig_HasTalent(tree, talent, rank)
	if not rank then rank = 1 end
	local _, _, _, _, r = GetTalentInfo(tree, talent)
	return r >= rank
end

local salvationbuffs = {
	"Spell_Holy_SealOfSalvation",
	"Spell_Holy_GreaterBlessingofSalvation"
}
function LazyPig_CheckSalvation()
	if LPCONFIG.SALVA ~= 1 and LPCONFIG.SALVA ~= 2 then
		return
	end
	if LPCONFIG.SALVA == 2 then
		local warriorTank = LazyPig_IsShieldEquipped() and LazyPig_PlayerClass("Warrior")
		local druidTank = LazyPig_IsBearForm()
		local paladinTank = LazyPig_HasRighteousFury()
		local shamanTank = LazyPig_IsShieldEquipped() and LazyPig_PlayerClass("Shaman") and LazyPig_HasTalent(shamanTankTree, shamanTankTalent, 2)
		if not (warriorTank or druidTank or paladinTank or shamanTank) then
			return
		end
	end
	local counter = 0
	while GetPlayerBuff(counter) >= 0 do
		local index, untilCancelled = GetPlayerBuff(counter)
		if untilCancelled ~= 1 then
			local texture = GetPlayerBuffTexture(index)
			if texture then
				local i = 1
				while salvationbuffs[i] do
					if string.find(texture, salvationbuffs[i]) then
						CancelPlayerBuff(index)
						UIErrorsFrame:Clear()
						UIErrorsFrame:AddMessage("Salvation Removed")
						return
					end
					i = i + 1
				end
			end
		end
		counter = counter + 1
	end
end

function LazyPig_RefreshCamera()
	if LPCONFIG.CAM then
		SetCVar("cameraDistanceMax", 50)
	else
		SetCVar("cameraDistanceMaxFactor", 1)
		SetCVar("cameraDistanceMax", 15)
	end
end

local manabuffs = {
	"Spell_Holy_SealOfWisdom",
	"Spell_Holy_GreaterBlessingofWisdom",
	"Spell_Holy_ArcaneIntellect",
	"Spell_Holy_MagicalSentry",
	"Spell_Holy_PrayerofSpirit",
	"Spell_Holy_DivineSpirit"
}
function LazyPig_CheckManaBuffs()
	if not LPCONFIG.REMOVEMANABUFFS or LazyPig_BG() then
		return
	end
	local counter = 0
	while GetPlayerBuff(counter) >= 0 do
		local index, untilCancelled = GetPlayerBuff(counter)
		if untilCancelled ~= 1 then
			local texture = GetPlayerBuffTexture(index)
			if texture then
				local i = 1
				while manabuffs[i] do
					if string.find(texture, manabuffs[i]) then
						CancelPlayerBuff(index)
						UIErrorsFrame:Clear()
						UIErrorsFrame:AddMessage("Intellect or Wisdom or Spirit Removed")
						return
					end
					i = i + 1
				end
			end
		end
		counter = counter + 1
	end
end

function LazyPig_ChatFrame_OnEvent(event)
	if event == "CHAT_MSG_LOOT" or event == "CHAT_MSG_MONEY" then
		local bijou = string.find(arg1 ,"Bijou")
		local coin = string.find(arg1 ,"Coin")
		local idol = string.find(arg1, "Idol")
		local scarab = string.find(arg1, "Scarab")
		local green_roll = greenrolltime > GetTime()
		local check_uncommon = LPCONFIG.SPAM_UNCOMMON and string.find(arg1 ,"1eff00")
		local check_rare = LPCONFIG.SPAM_RARE and string.find(arg1 ,"0070dd")
		local check_loot = LPCONFIG.SPAM_LOOT and (string.find(arg1 ,"9d9d9d") or string.find(arg1 ,"ffffff") or string.find(arg1 ,"Your share of the loot"))
		local check_money = LPCONFIG.SPAM_LOOT and string.find(arg1 ,"Your share of the loot")

		local check1 = string.find(arg1 ,"You")
		local check2 = string.find(arg1 ,"won") or string.find(arg1 ,"receive")
		local check3 = LPCONFIG.AQ and (idol or scarab)
		local check4 = LPCONFIG.ZG and (bijou or coin)
		local check5 = check1 and not check4 and not check3 and not green_roll or check2

		if not check5 and (check_uncommon or check_rare) or check_loot and not check1 or check_money then
			return
		end
	end

	if LPCONFIG.SPAM and arg2 and arg2 ~= GetUnitName("player") and (event == "CHAT_MSG_SAY" or event == "CHAT_MSG_CHANNEL" or event == "CHAT_MSG_YELL" or event == "CHAT_MSG_EMOTE" and not (IsGuildMate(arg2) or IsFriend(arg2))) then
		local time = GetTime()
		local index = ChatMessage["INDEX"]

		for blockindex,blockmatch in pairs(ChatMessage[index]) do
			local findmatch1 = (blockmatch + 70) > time --70s delay
			local findmatch2 = blockindex == arg1
			if findmatch1 and findmatch2 then
				return
			end
		end
		ChatMessage[index][arg1] = time
	end

    -- suppress BigWigs spam
	if LPCONFIG.SPAM and event == "CHAT_MSG_SAY" and string.find(arg1 or "" ,"^Casted %u[%a%s]+ on %u[%a%s]+") then
        return
    end

	-- suppress #showtooltip spam
	if string.find(arg1 or "" , "^#showtooltip") then
		return
	end

	-- suppress rested xp spam
	if string.find(arg1 or "", "=== .+ RESTED XP") and arg2 ~= UnitName("player") then
		return
	end

	Original_ChatFrame_OnEvent(event);
end

function ChatSpamClean()
	local time = GetTime()
	local index = ChatMessage["INDEX"]
	local newindex = nil

	if index == 1 then
		newindex = 2
	else
		newindex = 1
	end

	for blockindex,blockmatch in pairs(ChatMessage[index]) do
		if (blockmatch + 70) > time then
			ChatMessage[newindex][blockindex] = ChatMessage[index][blockindex]
		end
	end
	ChatMessage[index] = twipe(ChatMessage[index])
	ChatMessage["INDEX"] = newindex

	--DEFAULT_CHAT_FRAME:AddMessage("ChatSpamClean")
end

function LazyPig_Track_EFC(msg)
	if msg then
		msg = strlower(msg)

		local find0 = "captured "
		local find1 = "The "..UnitFactionGroup("player").." Flag"
		local find2 = " was picked up "
		local find3 = " was dropped "

		if string.find(msg, strlower(find1..find2)) then
			_, _, wsgefc = string.find(msg, strlower(find1..find2.."by (.+)%!"))
			--DEFAULT_CHAT_FRAME:AddMessage("ADD EFC - "..wsgefc)
		elseif string.find(msg, strlower(find1..find3)) or string.find(msg, strlower(find0..find1)) then
			wsgefc = nil
			--DEFAULT_CHAT_FRAME:AddMessage("DEL EFC")
		end
	end
end

function LazyPig_Target_EFC()
	ClearTarget()
	if wsgefc then
		TargetByName(wsgefc, true)
		UIErrorsFrame:Clear()
		if not UnitExists("target") then
			UIErrorsFrame:AddMessage("OUT OF RANGE - EFC - "..strupper(wsgefc))
		elseif strlower(GetUnitName("target")) == wsgefc then
			local class, classFileName = UnitClass("target")
			local color = RAID_CLASS_COLORS[classFileName]
			UIErrorsFrame:AddMessage(strupper(class.." - EFC - "..wsgefc), color.r, color.g, color.b)
		end
	end
end

function LazyPig_Duel_EFC()
	if GetRealZoneText() == "Warsong Gulch" then
		LazyPig_Target_EFC()
	else
		local duel = nil
		for i=1,STATICPOPUP_NUMDIALOGS do
			local frame = _G["StaticPopup"..i]
			if frame:IsShown() then
				if frame.which == "DUEL_REQUESTED" then
					duel = true
				end
			end
		end
		if duel_active or duel then
			CancelDuel()
		elseif UnitExists("target") and UnitIsFriend("target", "player") then
			StartDuel(GetUnitName("target"))
		end
	end
end

function LazyPig_HasRighteousFury()
	if not LazyPig_PlayerClass("Paladin") then return false end
	local counter = 0
	while GetPlayerBuff(counter) >= 0 do
		local index, untilCancelled = GetPlayerBuff(counter)
		if untilCancelled == 1 then
			local texture = GetPlayerBuffTexture(index)
			if texture and string.find(texture, "Spell_Holy_SealOfFury") then
				return true
			end
		end
		counter = counter + 1
	end
	return false
end
```

## LazyPig.xml
```xml
<Ui xmlns="http://www.blizzard.com/wow/ui/" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://www.blizzard.com/wow/ui/
C:\Projects\WoW\Bin\Interface\FrameXML\UI.xsd">
	<Script file="LazyPig.lua"/>
	<Script file="LazyPigMenu.lua"/>
	<Script file="LazyPigKeybindsFrame.lua"/>
	<Frame name="LazyPig">	
		<Scripts>
			<OnLoad>
				LazyPig_OnLoad()
			</OnLoad>
			<OnEvent>
				LazyPig_OnEvent(event)
			</OnEvent>
			<OnUpdate>
				LazyPig_OnUpdate(arg1)
			</OnUpdate>
		</Scripts>
		<Anchors>
			<Anchor point="TOP" relativeTo="PVPArenaTextString" relativePoint="BOTTOM" />
		</Anchors>
		<Size x="256" y="256"/>
		<Layers>
			<Layer level="ARTWORK">		
				<FontString name="LazyPigText" font="Fonts\FRIZQT__.TTF" outline="THICK" hidden="true">
					<Size x="512" y="0"/>
					<Anchors>
						<Anchor point="CENTER">
							<Offset x="0" y="90"/>
						</Anchor>
					</Anchors>
					<FontHeight val="18"/>
				</FontString>	
			</Layer>
		</Layers>
	</Frame>
	<GameTooltip name="LazyPig_Buff_Tooltip" hidden="true" inherits="GameTooltipTemplate">
		<Scripts>
			<OnLoad>
				this:SetOwner(WorldFrame, "ANCHOR_NONE")
			</OnLoad>
		</Scripts>
	</GameTooltip>
</Ui>
```

## LazyPigKeybindsFrame.lua
```lua
local _G = _G or getfenv(0)

local FontstringTables = {
	["Non-Configurable Keys"] = {
		[0] = "LazyPigNCE",
		[1] = { "LazyPig_kbfs000", "Sell Grey Items/Repairs:", "Hold Alt Key while Merchant's window is open" },
		[2] = { "LazyPig_kbfs001", "Repeatable Quest Auto-Complete:", "Hold Alt Key and finish quest once to record the steps." },
		[3] = { "LazyPig_kbfs002", "Quest Auto-PickUp/Auto-Complete:", "Hold Alt Key to Pickup/Complete quests." },
	},

	["Special Key-Combinations"] = {
		[0] = "LazyPigSKCB",
		[1] = { "LazyPig_kbfs010", "Follow:", "CTRL-SHIFT" },
		[2] = { "LazyPig_kbfs011", "Inspect Player/Bid Auction:", "ALT-SHIFT" },
		[3] = { "LazyPig_kbfs012", "Send Mail/Create Auction:", "CTRL-ALT" },
		[4] = { "LazyPig_kbfs013", "Confirm Popup/Buy Auction:", "CTRL-ALT" },
		[5] = { "LazyPig_kbfs014", "Initiate-Accept Trade:", "CTRL-ALT" },
	},

	["Configurable Keys"] = {
		[0] = "LazyPigCKB",
		[1] = { "LazyPig_kbfs020", "Logout:", "" },
		[2] = { "LazyPig_kbfs021", "Unstuck", "" },
		[3] = { "LazyPig_kbfs022", "Reload UI:", "" },
		[4] = { "LazyPig_kbfs023", "Target WSG EFC/Duel Request-Cancel", "" },
		[5] = { "LazyPig_kbfs024", "Drop WSG Flag/Remove Slow Fall Buff", "" },
	},
}


local function FontstringGroup(hParent, offsetX, offsetY, sTitle, tCheck, tCol1, tCol2, bKB)
	local frame = CreateFrame("Frame", tCheck[0], hParent)
	frame:SetPoint("TOPLEFT", hParent, "TOPLEFT", offsetX, offsetY)
	frame:SetWidth(11)
	frame:SetHeight(11)

	local fs_title = frame:CreateFontString(nil, "ARTWORK", "GameFontNormalSmall")
	fs_title:SetPoint("TOPLEFT", frame, "TOPLEFT", 0, 0)
	fs_title:SetTextColor(1, 1, 1, 1)
	fs_title:SetText(sTitle)

	frame.fs_title = fs_title

	frame.fst = {}
	frame.fsc = {}
	
	local max_width = { 0, nil }

	for k,v in ipairs(tCheck) do
		local fst = frame:CreateFontString(v[1], "ARTWORK", "GameFontNormalSmall")
		fst:SetPoint("TOPLEFT", frame, "BOTTOMLEFT", 8, -(4+(k-1)*10))
		fst:SetText(v[2])
		fst:SetTextColor(tCol1[1], tCol1[2], tCol1[3], tCol1[4])
		local temp = fst:GetStringWidth()
		if temp > max_width[1] then 
			max_width[1] = temp
			max_width[2] = fst
		end

		frame.fst[k] = fst
	end
	for k,v in ipairs(tCheck) do
		local fsname
		if not bKB then
			fsname = v[1].."Content"
		else
			fsname = "LP_KB" .. tostring(k)
		end
		local fsc = frame:CreateFontString(fsname, "ARTWORK", "GameFontNormalSmall")
		fsc:SetPoint("LEFT", _G[v[1]], "LEFT", 16 + max_width[1], 0)
		fsc:SetText(v[3])
		fsc:SetTextColor(tCol2[1], tCol2[2], tCol2[3], tCol2[4])

		frame.fsc[k] = fsc
	end
end

local function ShowBindings(bind, fs, desc)
	local bind1, bind2 = GetBindingKey(bind)
	local fsl = _G[fs]

	local printout = nil
	if bind1 and bind2 then
		printout = "[" .. bind1 .. "/" .. bind2 .. "]"
	elseif bind1 then
		printout = "[" .. bind1 .. "]"
	elseif bind2 then
		printout = "[" .. bind2 .. "]"
	elseif desc then
		printout = "[" .. desc .. "]"
	else
		printout = "none"
		fsl:SetTextColor(1,1,1,1)
	end
	fsl:SetText(printout)
end

function LazyPig_CreateKeybindsFrame()
	-- Keybinds Frame
	local frame = CreateFrame("Frame", "LazyPigKeybindsFrame", LazyPigOptionsFrame)
	frame:SetWidth(600)
	frame:SetHeight(175)
	frame:SetFrameLevel(LazyPigOptionsFrame:GetFrameLevel() + 2)
	frame:SetPoint("TOP", LazyPigOptionsFrame, "BOTTOM", 0, 10)
	frame:SetBackdrop({
		bgFile = "Interface\\Buttons\\WHITE8x8",
		edgeFile = "Interface\\DialogFrame\\UI-DialogBox-Border",
		tile = true,
		tileSize = 32,
		edgeSize = 32,
		insets = { left = 11, right = 12, top = 12, bottom = 11 }
	})
	frame:SetBackdropColor(0, 0, 0, .8)
	frame:SetMovable(true)
	frame:EnableMouse(true)
	frame:RegisterForDrag("LeftButton")
	frame:Hide()
	frame:SetScript("OnShow", function()
		ShowBindings("LOGOUT", "LP_KB1", "CTRL+ALT+SHIFT")
		ShowBindings("UNSTUCK", "LP_KB2")
		ShowBindings("RELOAD", "LP_KB3")
		ShowBindings("DUEL", "LP_KB4")
		ShowBindings("WSGDROP", "LP_KB5")
	end)
	frame:SetScript("OnMouseDown", function()
		if arg1 == "LeftButton" and not LazyPigOptionsFrame.isMoving then
			LazyPigOptionsFrame:StartMoving()
			LazyPigOptionsFrame.isMoving = true
		end
	end)
	frame:SetScript("OnMouseUp", function()
		if arg1 == "LeftButton" and LazyPigOptionsFrame.isMoving then
			LazyPigOptionsFrame:StopMovingOrSizing()
			LazyPigOptionsFrame.isMoving = false
		end
	end)
	frame:SetScript("OnHide", function()
		if LazyPigOptionsFrame.isMoving then
			LazyPigOptionsFrame:StopMovingOrSizing()
			LazyPigOptionsFrame.isMoving = false
		end
	end)

	-- MenuTitle Frame
	local texture_title = frame:CreateTexture("LazyPigKeybindsFrameTitle")
	texture_title:SetTexture("Interface\\DialogFrame\\UI-DialogBox-Header")
	texture_title:SetWidth(266)
	texture_title:SetHeight(58)
	texture_title:SetPoint("CENTER", LazyPigKeybindsFrame, "TOP", 0, -20)

	frame.texture_title = texture_title

	-- MenuTitle FontString
	local fs_title = frame:CreateFontString(nil, "ARTWORK", "GameFontNormalSmall")
	fs_title:SetPoint("CENTER", frame.texture_title, "CENTER", 0, 12)
	fs_title:SetText("LazyPig Keybinds")

	frame.fs_title = fs_title

	local st = "Non-Configurable Keys"
	frame.fsgroup_NCE = FontstringGroup(frame, 20, -25, st, FontstringTables[st], {1, .81, 0}, {1, 1, 1})
	st = "Special Key-Combinations"
	frame.fsgroup_SKCB = FontstringGroup(frame, 20, -85, st, FontstringTables[st], {1, .81, 0}, {.8, .1, .1})
	st = "Configurable Keys"
	frame.fsgroup_CKB = FontstringGroup(frame, 255, -85, st, FontstringTables[st], {1, .81, 0}, {.8, .1, .1}, true)
end
```

## LazyPigMenu.lua
```lua
LazyPigOptions = {
	{
		text = "Green Items Roll [Ctrl + Alt]",
		exclusive = true,
		checkBoxes = {
			{ text = NEED, var = "GREEN", value = 1, tooltip = NEED },
			{ text = GREED, var = "GREEN", value = 2, tooltip = GREED },
			{ text = PASS, var = "GREEN", value = 0, tooltip = PASS },
		},
	},
	{
		text = "Zul'Gurub Roll Automation",
		exclusive = true,
		checkBoxes = {
			{ text = NEED, var = "ZG", value = 1, tooltip = NEED },
			{ text = GREED, var = "ZG", value = 2, tooltip = GREED },
			{ text = PASS, var = "ZG", value = 0, tooltip = PASS },
		},
	},
	{
		text = "Molten Core Roll Automation",
		exclusive = true,
		checkBoxes = {
			{ text = NEED, var = "MC", value = 1, tooltip = NEED },
			{ text = GREED, var = "MC", value = 2, tooltip = GREED },
			{ text = PASS, var = "MC", value = 0, tooltip = PASS },
		},
	},
	{
		text = "AQ Idols + Scarabs Automation",
		exclusive = true,
		checkBoxes = {
			{ text = NEED, var = "AQ", value = 1, tooltip = NEED },
			{ text = GREED, var = "AQ", value = 2, tooltip = GREED },
			{ text = PASS, var = "AQ", value = 0, tooltip = PASS },
		},
	},
	{
		text = "Black Morass Corrupted Sand Automation",
		exclusive = true,
		checkBoxes = {
			{ text = NEED, var = "SAND", value = 1, tooltip = NEED },
			{ text = GREED, var = "SAND", value = 2, tooltip = GREED },
			{ text = PASS, var = "SAND", value = 0, tooltip = PASS },
		},
	},
	{
		text = "Emerald Sanctum Roll Automation",
		exclusive = true,
		checkBoxes = {
			{ text = NEED, var = "ES_SHARDS", value = 1, tooltip = NEED },
			{ text = GREED, var = "ES_SHARDS", value = 2, tooltip = GREED },
			{ text = PASS, var = "ES_SHARDS", value = 0, tooltip = PASS },
		},
	},
	{
		text = "Naxxramas Roll Automation",
		exclusive = true,
		checkBoxes = {
			{ text = NEED, var = "NAXX", value = 1, tooltip = NEED },
			{ text = GREED, var = "NAXX", value = 2, tooltip = GREED },
			{ text = PASS, var = "NAXX", value = 0, tooltip = PASS },
		},
	},
	{
		text = "Salvation Remover",
		exclusive = true,
		checkBoxes = {
			{ text = ALWAYS, var = "SALVA", value = 1, tooltip = ALWAYS, setFunc = LazyPig_CheckSalvation },
			{ text = "Smart", var = "SALVA", value = 2, tooltip = "Smart", tooltipSub = "Auto remove if:\nYou are Warrior and have shield equipped,\nYou are Shaman and have shield equipped and have Spirit Armor talent rank 2,\nYou are Druid in Bear Form,\nYou are Paladin with Righteous Fury.", setFunc = LazyPig_CheckSalvation },
		},
	},
	{
		text = "Mana Buffs Remover",
		checkBoxes = {
			{ text = ALWAYS, var = "REMOVEMANABUFFS", tooltip = "Auto remove Blessing of Wisdom / Arcane Intellect / Prayer of Spirit. Disabled in battlegrounds.", setFunc = LazyPig_CheckManaBuffs },
		},
	},
	{
		text = "Group Invite Accept Rules",
		checkBoxes = {
			{ text = "GuildMates", var = "GINV", tooltip = "Auto accept invites from guild members." },
			{ text = "Friends", var = "FINV", tooltip = "Auto accept invites from friends." },
			{ text = "Strangers", var = "SINV", tooltip = "Auto accept invites from strangers." },
			{ text = "Idle while in BG or Queue", var = "DINV", tooltip = "Do not auto accept invites while in battleground or queue." }
		},
	},
	{
		text = "Battlegrounds Automation",
		checkBoxes = {
			{ text = "Enter Battleground", var = "EBG", tooltip = "Enter battleground as soon as the queue popup." },
			{ text = "Leave Battleground", var = "LBG", tooltip = "Leave battleground as soon as it's finished." },
			{ text = "Queue Battleground", var = "QBG", tooltip = "Join first available queue as soon as the battle master is right-clicked." },
			{ text = "Auto Release", var = "RBG", tooltip = "Auto release spirit in battlegrounds." },
			{ text = "Leader Queue Announce", var = "AQUE", tooltip = "Auto announce joining battleground queue as a raid leader." },
			{ text = "Block BG Quest Sharing", var = "SBG", tooltip = "Auto decline shared quests while in battlegrounds." }
		},
	},
	{
		text = "Chat Filter",
		checkBoxes = {
			{ text = "Players' Spam", var = "SPAM", tooltip = "Hide players' spam messages." },
			{ text = "Uncommon Roll", var = "SPAM_UNCOMMON", tooltip = "Hide uncommon (green) loot roll messages." },
			{ text = "Rare Roll", var = "SPAM_RARE", tooltip = "Hide rare (blue) loot roll messages." },
			{ text = "Poor-Common Loot", var = "SPAM_LOOT", tooltip = "Hide poor and common (grey and white) loot roll messages." },
			{ text = "LazyPig Messages", var = "ROLLMSG", tooltip = "Show LazyPig auto roll messages." }
		},
	},
	{
		text = "World Chat Mute",
		checkBoxes = {
			{ text = "Dungeons", var = "WORLDDUNGEON", tooltip = "Mute world chat while in dungeons.", setFunc = LazyPig_ZoneCheck },
			{ text = "Raids", var = "WORLDRAID", tooltip = "Mute world chat while in raids.", setFunc = LazyPig_ZoneCheck },
			{ text = "Battlegrounds", var = "WORLDBG", tooltip = "Mute world chat while in battlegrounds.", setFunc = LazyPig_ZoneCheck },
			{ text = "Mute Permanently", var = "WORLDUNCHECK", tooltip = "Mute world chat for good...", setFunc = LazyPig_ZoneCheck }
		},
	},
	{
		text = MISCELLANEOUS,
		checkBoxes = {
			{ text = "Summon Auto Accept", var = "SUMM", tooltip = "Summon Auto Accept", tooltipSub = "If checked summons will be accepted before they expire." },
			{ text = "Loot Window Auto Position", var = "LOOT", tooltip = "Loot Window Auto Position", tooltipSub = "Position the loot window under the mouse cursor."},
			{ text = "Improved Right Click", var = "RIGHT", tooltip = "Improved Right Click", tooltipSub = "Right Click to drag and drop items into Mail, Trade, Auction Frames.", setFunc = LazyPig_MailtoCheck },
			{ text = "Easy Stack Split/Merge", var = "SHIFTSPLIT", tooltip = "Easy stack split/merge", tooltipSub = "Shift + Right Click on item to start, hold Control or Alt to change split amount, Shift + Right Click on item again to finish.", setFunc = LazyPig_MailtoCheck },
			{ text = "Extended Camera Distance", var = "CAM", tooltip = "Extended Camera Distance", tooltipSub = "Extend the camera distance to its maximum.", setFunc = LazyPig_RefreshCamera },
			{ text = "Special Key Combinations", var = "SPECIALKEY", tooltip = "Special Key Combinations", tooltipSub = "Activate the Special Key Combinations, click Keybinds button for more details." },
			{ text = "Duel Auto Decline", var = "DUEL", tooltip = "Auto decline duels (hold Shift to bypass)." },
			{ text = "Instance Resurrection Auto Accept", var = "REZ", tooltip = "Instance Resurrection Auto Accept", tooltipSub = "Auto accept resurrection in raids, dungeons and battlegrounds if player resurrecting you is out of combat." },
			{ text = "Gossip Auto Processing", var = "GOSSIP", tooltip = "Gossip Auto Processing (hold Shift to bypass)", tooltipSub = "Skip gossip choises from innkeepers, fly masters etc." },
			{ text = "Auto Dismount", var = "DISMOUNT", tooltip = "Auto Dismount", tooltipSub = "Auto dismount when it's required by another action." },
			{ text = "Auto Stance", var = "AUTOSTANCE", tooltip = "Auto Stance", tooltipSub = "Automatically change to required warrior stance/druid form on spell cast." },
		},
	},
}

function LazyPig_CreateOptionsFrame()
	-- Option Frame
	local frame = CreateFrame("Frame", "LazyPigOptionsFrame", UIParent)
	tinsert(UISpecialFrames,"LazyPigOptionsFrame")
	frame:SetFrameStrata("DIALOG")
	frame:SetWidth(500)
	frame:SetHeight(670)
	frame:SetPoint("CENTER", UIParent, 0, 80)
	frame:SetBackdrop({
		bgFile = "Interface\\Buttons\\WHITE8x8",
		edgeFile = "Interface\\DialogFrame\\UI-DialogBox-Border",
		tile = true,
		tileSize = 32,
		edgeSize = 32,
		insets = { left = 11, right = 12, top = 12, bottom = 11 }
	})
	frame:SetBackdropColor(0, 0, 0, .8)
	frame:SetMovable(true)
	frame:EnableMouse(true)
	frame:SetClampedToScreen(false)
	frame:RegisterForDrag("LeftButton")
	frame:Hide()
	frame:SetScript("OnMouseDown", function()
		if arg1 == "LeftButton" and not this.isMoving then
			this:StartMoving();
			this.isMoving = true;
		end
	end)
	frame:SetScript("OnMouseUp", function()
		if arg1 == "LeftButton" and this.isMoving then
			this:StopMovingOrSizing();
			this.isMoving = false;
		end
	end)
	frame:SetScript("OnHide", function()
		if this.isMoving then
			this:StopMovingOrSizing();
			this.isMoving = false;
		end
	end)

	-- MenuTitle Frame
	local texture_title = frame:CreateTexture("LazyPigOptionsFrameTitle")
	texture_title:SetTexture("Interface\\DialogFrame\\UI-DialogBox-Header", true);
	texture_title:SetWidth(296)
	texture_title:SetHeight(58)
	texture_title:SetPoint("CENTER", frame, "TOP", 0, -20)

	frame.texture_title = texture_title

	-- MenuTitle FontString
	local fs_title = frame:CreateFontString(nil, "ARTWORK", "GameFontNormalSmall")
	local LP_VERSION = GetAddOnMetadata("_LazyPig", "Version")
	fs_title:SetPoint("CENTER", frame.texture_title, "CENTER", 0, 12)
	fs_title:SetText("LazyPig Options")
	
	local versionText = frame:CreateFontString("$parentVersionText", "ARTWORK", "GameFontNormalSmall")
	versionText:SetPoint("TOPLEFT", frame, 20, -20)
	versionText:SetText("version: "..LP_VERSION)

	frame.fs_title = fs_title

	-- Close Setting Window Button
	local btn_close = CreateFrame("Button", "LazyPigOptionsFrameCloseButton", frame, "UIPanelCloseButton")
	btn_close:SetPoint("TOPRIGHT", frame, "TOPRIGHT", -12, -12)
	btn_close:SetWidth(32)
	btn_close:SetHeight(32)
	
	frame.btn_close = btn_close

	frame.btn_close:SetScript("OnClick", function()
		this:GetParent():Hide()
			LazyPigKeybindsFrame:Hide()
	end)

	-- Keybinds Window Button
	local btn_keybinds = CreateFrame("Button", "LazyPigOptionsFrameKeibindsButton", frame, "UIPanelButtonTemplate2")
	btn_keybinds:SetPoint("BOTTOMRIGHT", frame, "BOTTOMRIGHT", -33, 18)
	btn_keybinds:SetWidth(120)
	btn_keybinds:SetHeight(20)
	btn_keybinds:SetText(KEY_BINDINGS)
	btn_keybinds:SetTextColor(1, 1, 1)
	frame.btn_keybinds = btn_keybinds

	frame.btn_keybinds:SetScript("OnClick", function()
		if LazyPigKeybindsFrame:IsShown() then
			LazyPigKeybindsFrame:Hide()
		else
			LazyPigKeybindsFrame:Show()
		end
	end)

	local height = 16
	local insetLeft = 30
	local insetTop = -32
	local columnWidth = 240
	local offsetX, offsetY = insetLeft, insetTop
	local index = 1
	for i = 1, getn(LazyPigOptions) do
		if i == 10 then
			offsetX, offsetY = insetLeft + columnWidth, insetTop
		end
		
		-- Check box group title
		local fontString = frame:CreateFontString(nil, "ARTWORK", "GameFontNormalSmall")
		fontString:SetPoint("TOPLEFT", frame, "TOPLEFT", offsetX, offsetY - 10)
		fontString:SetTextColor(1, 1, 1, 1)
		fontString:SetText(LazyPigOptions[i].text)
		
		offsetY = offsetY - height - 5
		
		for j = 1, getn(LazyPigOptions[i].checkBoxes) do
			local checkBox = CreateFrame("CheckButton", "$parentCheckBox"..index, frame, "UICheckButtonTemplate")
			local checkBoxText = _G[checkBox:GetName().."Text"]
			
			checkBox:SetPoint("TOPLEFT", frame, "TOPLEFT", offsetX + 5, offsetY)
			checkBox:SetWidth(22)
			checkBox:SetHeight(22)
			checkBoxText:SetText(LazyPigOptions[i].checkBoxes[j].text)
			checkBox.textR, checkBox.textG, checkBox.textB = checkBoxText:GetTextColor()
			
			-- Makes text clickable
			checkBox:SetHitRectInsets(0, -(checkBoxText:GetWidth() + 5), 0, 0)
			
			LazyPigOptions[i].checkBoxes[j].frame = checkBox
			checkBox.tooltip = LazyPigOptions[i].checkBoxes[j].tooltip
			checkBox.tooltipSub = LazyPigOptions[i].checkBoxes[j].tooltipSub
			checkBox.var = LazyPigOptions[i].checkBoxes[j].var
			checkBox.value = LazyPigOptions[i].checkBoxes[j].value
			checkBox.exclusive = LazyPigOptions[i].exclusive
			checkBox.checkBoxes = LazyPigOptions[i].checkBoxes
			checkBox.setFunc = LazyPigOptions[i].checkBoxes[j].setFunc

			checkBox:SetScript("OnShow", function()
				local value = this.value or true
				if LPCONFIG[this.var] == value then
					this:SetChecked(true)
					if this.exclusive then
						for _, data in pairs(this.checkBoxes) do
							if data.frame ~= this then
								data.frame:SetChecked(false)
							end
						end
					end
				else
					this:SetChecked(false)
				end
			end)

			checkBox:SetScript("OnClick", function()
				if this.exclusive then
					for _, data in pairs(this.checkBoxes) do
						if data.frame ~= this then
							data.frame:SetChecked(false)
						end
					end
				end
				local value = this.value or true
				if type(value) == "boolean" then
					LPCONFIG[this.var] = not LPCONFIG[this.var]
					-- this:SetChecked(LPCONFIG[this.var])
				elseif type(value) == "number" then
					LPCONFIG[this.var] = this:GetChecked() and this.value or nil
					-- this:SetChecked(LPCONFIG[this.var] == 0 and true or LPCONFIG[this.var])
				end
				if this.setFunc then
					this.setFunc()
				end
				PlaySound("igMainMenuOptionCheckBoxOn")
			end)

			checkBox:SetScript("OnEnter", function()
				_G[this:GetName().."Text"]:SetTextColor(1, 1, 1)
				if this.tooltip then
					GameTooltip:SetOwner(this, "ANCHOR_TOPRIGHT")
					-- GameTooltip:SetBackdropColor(.01, .01, .01, .91)
					GameTooltip:SetText(this.tooltip, nil, nil, nil, 1, true)
					if this.tooltipSub then
						GameTooltip:AddLine(this.tooltipSub, 1, 1, 1, true)
					end
					GameTooltip:Show()
				end
			end)

			checkBox:SetScript("OnLeave", function()
				_G[this:GetName().."Text"]:SetTextColor(this.textR, this.textG, this.textB)
				GameTooltip:Hide()
			end)

			offsetY = offsetY - height
			index = index + 1
		end
	end
end
```

## Readme.txt
```txt
Updated for Turtle WoW Expansion mounts and Riding Turtle - autodismount


This mod is mostly for lazy people:


PASSIVE ENHANCEMENTS:
AUTO DISMOUNT 				- when casting spell or interacting with flightmaster(aq40 mounts included)
AUTO ZG ROLL 				- on bijou, coins(avaliable 4 modes: off, pass, greed, need)
LOOT WINDOW AUTO POSITION		- under the cursor when looting
GOSSIP AUTO PROCESSING			- when talking to NPC(taxi, battlemaster, innkeeper, vendors)
WORLD CHAT MUTE				- mute /world channel in raid/dungeon/bg zone also filter repeated messages(30s default ban time)
IMPROVED RIGHT CLICK			- if trade/auction/mail window is active, right click on container item will drag and drop it into trade/auction_create/send_mail window


ACTIVE ENHANCEMENTS(require shift modifier key press):
SHIFT SPLIT/MERGE			- shift + right will split stacked items
AUTO GREY SELL/REPAIR			- press shift when merchant frame is open
REPEATABLE QUESTS AUTOCOMPLETE 		- token quests like: AD(scourge stones,insignias), ZG(coins), BG(marks), Thorium Shells (v1.7changelog)

MINOR ENHANCEMENTS:
GROUP AUTO ACCEPT 			- 3 modes (GUILDMATES, GUILDMATES>FRIENDS, EVERYONE)
SUMMON AUTO ACCEPT 	  		- accept 2 seconds before confirm time expire
INSTANCE RESURECTION ACCEPT OOC		- instant accept OOC(ouf of combat)
EXTENDED CAMERA DISTANCE		- increase max camera distance up to 50y


SPECIAL KEY COMBINATIONS
alt+ctrl+shift				- logout

ctrl+shift				- follow

alt+shift 				- inspect, click button: bid_auction

alt+ctrl				- initiate/accept trade with other player, 
					  confirm popups: group_invite/bg_entry/release_spirit/recover_corpse/summon etc etc,
					  click button: send_mail/create_auction/buyout_auction/accept quest etc etc,
					  roll: on greenies other clolors are ignored, 3 modes (NEED,GREEN,PASS)





typing /lp    - customize addon functionality(most of the functions can be toggled)

Some of the addon's functions may not work if you're using not en/us localized client also bongos and oskin may cause some small issues. 


Addons with similar functionality you don't need anymore
- Auto Profit
- Ez Dismount
- Automaton
- Quick Loot
- Block Salation
- MailTo -> If you gonna use it anyways be sure to disable _LazyPig features(Improved Right Click and Shift Split/Merge) to avoid override.

New in v1.7 
Added new feature record/replay for repeatable quests that allows you to complete them extremely fast.
To record just hold down the Shift key and complete the repeatable quest, after that talk to the NPC again and all
the previous actions will be auto replayed, remember not to release the Shift key.
New in v1.8
Added split feature
- shift + right click will split stack
New in v1.9
Added auto roll option for zg rep items: bijous, coins 
New in 2.3
- added easy to use grey sell feature, just press shift key when merchant frame is open
- thorium shells exchange quest now can be automated(v1.7 changelog)
- added auto queue bg feature
New in 2.5
- resolved problem with dismounting when more than 16 buffs is active(tooltip instead of texture scanning)
- fixed all issues with the auto completing of repeatable quests (bg marks)
- fixed bg auto leave feature (did not trigger sometimes)
New in 2.7
- added new key combinations
- improved right click on container items
- added auto repair function(performed when merchant frame is visible and Shift key pressed)
- improved quest/gossip auto processing
New in 2.8
- improved Shift Split(see v1.8 changelog)
New in 2.83
- removed /lps command replaced by easy to use intuitive split mechanism(to enable split feature you need to type /lp 6 - if disabled)
New in 2.88
- added Alt+Ctrl green roll key combinations
New in 3.0
- added "Mute World" chat functionality, available 3 modes (RAID>DUNGEON>BG, RAID>DUNGEON, RAID) after leaving specified zone /world chat will be unmuted, function disabled as default type /lp for details
New in 3.50
- greatly improved split mechanism, check video
New in 4.00
- added GUI to acces type /lp
New in 4.40
- added keybindings
- added spam filter
- added salvation remover for tanks
- fixed many small bugs
New in 4.50
- couple fixes
- added drop WSG Flag bindig
cheers Ogrisch.
New in 4.52
- added find wsg efc binding
- improved REPEATABLE QUESTS AUTOCOMPLETE for wsg and arathi(if you're lvl 60 you will always complete lvl 60 version no matter which one you choose)
Some of the addon's functions may not work if you're using not en/us localized client also bongos and oskin may cause some small issues. 
New in 4.56
- some minor fixes
- resurrection auto accept now available only in instances
New in 5.00
- Block Battleground Quest Share feature(no more Empty Stables)
- Brand New GUI made by mrmr thx bro
New in 5.01
-added auto mana buff remover (Intellect, Spirit, Wisdom).
```

