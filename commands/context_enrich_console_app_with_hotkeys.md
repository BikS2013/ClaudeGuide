**Important** Before you do anything else 
**Ask me to give you the name of the server to enrich** and then:

### 4. Console Standard Features 
Implement console commands for the server that allow users to:
  - Type 'c' + Enter to clear console
  - Type 'f' + Enter to freeze/unfreeze output
  - Type 'v' + Enter to toggle verbose mode
  - Type 'h' + Enter to show help

Indicative console output:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   Type a letter and press Enter:
   • c ↵  : Clear console
   • f ↵  : Freeze/Unfreeze output
   • v ↵  : Toggle verbose mode
   • h ↵  : Show this help
   • Ctrl+C : Exit application
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Requirements:
  1. Console commands Must work with 'npm run dev' (nodemon)
  2. Use readline interface, not raw terminal mode
  3. Don't block or interfere with server operations
  4. Gracefully handle non-TTY environments, allowing commands to work
  5. Setup commands only after server successfully starts
  6. Include a standalone test script to verify functionality
  7. The console hot keys must work when running npm run dev with nodemon - handle the non-TTY environment   appropriately
  8. The C hot key must clear the console to allow user to select all (cmd-a) for the new messages without anything else in the way.

**Important** Before you proceed with the actual implementation, present me the concept and ask me to approve it.