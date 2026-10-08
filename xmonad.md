-- ~/.xmoand/xmonad.hs
import XMonad
import XMonad.Hooks.DynamicLog --xmobar
import XMonad.Hooks.ManageDocks --xmobar docks
import XMonad.Hooks.ManageHelpers (isFullscreen, doFullFloat) --search & action
import XMonad.Hooks.EwmhDesktops --xmobar display

import XMonad.Layout.NoBorders (smartBorders) --layout-fix-tool
import qualified XMonad.Layout.Tabbed as Tab
import XMonad.Layout.WindowNavigation (windowNavigation, Direction2D(..), Navigate(..))
import XMonad.Layout.BoringWindows (boringAuto)
import XMonad.Layout.SubLayouts (subLayout, GroupMsg(..), onGroup)
import XMonad.Layout.Simplest (Simplest(..))

import XMonad.Prompt
import XMonad.Prompt.Shell (shellPrompt)
import XMonad.Prompt.Ssh (sshPrompt)

import XMonad.Actions.GridSelect (goToSelected)
import Data.Default (def)
import XMonad.Actions.SpawnOn (manageSpawn, spawnHere)

import XMonad.Util.Run (spawnPipe) --start and find 'stdin handle'
import XMonad.Util.EZConfig (additionalKeysP, removeKeysP, checkKeymap) --use 'M-S-f' define custom keyboard shortcut.
import XMonad.Util.Loggers

import System.IO (hPutStrLn) --write xmobar
import qualified XMonad.StackSet as W
import Control.Monad (when)

main :: IO ()
main = do
  xmproc <- spawnPipe "xmobar ~/.config/xmobar/xmobarrc"
  hPutStrLn xmproc "starting..." --log
  xmonad $ ewmh $ docks def
    { modMask = mod4Mask -- win-->alt
    , terminal = "alacritty"
    , borderWidth = 2
    , normalBorderColor = "#2A2A30"
    , focusedBorderColor = "#39C5BB"
    , startupHook = checkKeymap cfg myKeys
    , manageHook = manageHook def
                   <+> manageSpawn
                   <+> (isFullscreen --> doFullFloat)
    , layoutHook = myLayout
    , logHook = dynamicLogWithPP xmobarPP
      { ppOutput = hPutStrLn xmproc
      , ppTitle = xmobarColor "#7FDBDA" "" . shorten 39
      , ppCurrent = xmobarColor "#39C5BB" "" . wrap "[" "]"
      , ppHidden = xmobarColor "#D9E0E8" ""
      , ppLayout = xmobarColor "#f9e2af" "" . drop 7
      , ppUrgent = xmobarColor "#f06292" "" . wrap "!" "!"
      , ppSep = " | "
      , ppExtras = [ battery ]
      }
    }
    `removeKeysP` ["M-<Return>"]
    `additionalKeysP` myKeys
    where
      cfg = additionalKeysP (def { modMask = mod4Mask }) myKeys

--Layout
myLayout =
    avoidStruts
  $ smartBorders
  $ boringAuto
  $ windowNavigation
  $ Tab.addTabs Tab.shrinkText myTabTheme
  $ subLayout [] Simplest
  $ Tall 1 (3/100) (1/2)
    ||| Full
    ||| Mirror (Tall 1 (3/100) (1/2))
myTabTheme :: Tab.Theme
myTabTheme = Tab.def
  { Tab.activeColor = "#39c5bb"
  , Tab.inactiveColor = "#1b1b1f"
  , Tab.activeTextColor = "#1b1b1f"
  , Tab.inactiveTextColor = "#d9e0e8"
  , Tab.activeBorderColor = "#39c5bb"
  , Tab.inactiveBorderColor = "#2a2a30"
  , Tab.fontName = "xft:JetBrains Mono:size=10"
  , Tab.decoHeight = 20
  }
myXPConfig :: XPConfig
myXPConfig = def
  { font = "xft:JetBrains Mono:size=11"
  , bgColor = "#1b1b1f"
  , fgColor = "#d9e0e8"
  , borderColor = "#39c5bb"
  , promptBorderWidth = 1
  , height = 24
  }

--Keyboard shortcut
myKeys :: [(String, X ())]
myKeys =
  [ ("M-c",          spawnHere "alacritty" )
  , ("M-S-f",        spawn "firefox")
  , ("M-p",          shellPrompt myXPConfig)
  , ("M-S-p",        sshPrompt myXPConfig)
  , ("M-g",          goToSelected def)
  , ("M-b",          sendMessage ToggleStruts)
  , ("M-<Return>",   sendMessage NextLayout)
  , ("M-n",     sendMessage $ Go L)
  , ("M-e",     sendMessage $ Go R)
  , ("M-u",     sendMessage $ Go U)
  , ("M-i",     sendMessage $ Go D)
  , ("M-C-m",        withFocused $ sendMessage . MergeAll)
  , ("M-S-o",        withFocused $ sendMessage . UnMerge)
  , ("M-C-.",   onGroup W.focusDown')
  , ("M-C-,",   onGroup W.focusUp')
  , ("M-q",     spawn "xmonad --recompile && xmonad --restart")
  ]
