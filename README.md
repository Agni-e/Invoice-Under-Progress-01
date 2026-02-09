Root Cause Analysis
I've found multiple critical bugs across both parts. Let me walk through each flow:

🔴 CRITICAL BUG #1: Two Missing Functions
parseClientName() and cleanPublicationName() are called everywhere but never defined. This crashes the entire system immediately.

🔴 CRITICAL BUG #2: processEmailQueueWrapper — Trigger ID Mismatch
JavaScript

// In createDelayedTrigger():
const triggerId = String(trigger.getUniqueId());  // Returns a NUMBER like 4519703072

// In processEmailQueueWrapper():
const triggerId = e.triggerUid.toString();  // Returns the SAME number BUT...

// The PROBLEM: e.triggerUid is a NUMBER, trigger.getUniqueId() is also a NUMBER
// BUT sometimes Google returns them in different formats (string vs number comparison fails)
🔴 CRITICAL BUG #3: Queue Key Uses Date — But Dates Are Unreliable
JavaScript

const submittedDate = formatDateForCompare(rowData.submittedDate);
const queueKey = clientEmail + '|' + submittedDate + '|' + status;
// If submittedDate is empty or a Date object that formats differently, 
// the key won't match across calls
🔴 CRITICAL BUG #4: sendConsolidatedReminder Generates NEW Invoice Numbers (Ignoring Consolidation)
JavaScript

// In processQueueEntry():
const consolidatedInvoice = consolidateInvoiceNumbers(articles, entry.status);
// ↑ This sets the invoice number correctly

// BUT then in sendConsolidatedReminder():
if (!article.invoiceNum) {
  article.invoiceNum = generateInvoiceNumber(article.publication);  // OVERWRITES!
}
// AND for multi-article:
invoiceNum = generateInvoiceNumber(publication);  // GENERATES A NEW ONE!
The send functions ignore the consolidated invoice number and generate fresh ones.

🔴 CRITICAL BUG #5: sendConsolidatedConfirmation Same Problem
JavaScript

const invoiceNumber = isSingle ? 
  (articles[0].invoiceNum || generateInvoiceNumber(publication)) : 
  generateInvoiceNumber(publication);  // ALWAYS generates new for multi-article!
🔴 CRITICAL BUG #6: consolidateInvoiceNumbers Regex Crash
JavaScript

var seqA = parseInt(a.invoice.match(/-(\d{4})-/)[1]);
// If invoice format doesn't have -NNNN- pattern, match() returns null
// null[1] = CRASH
🔴 CRITICAL BUG #7: handlePaidStatus Doesn't Cancel Pending Reminders
When status changes from Reminder → Paid, the pending reminder trigger still fires, potentially sending BOTH a reminder AND confirmation.

🔴 CRITICAL BUG #8: getRowData Returns Stale Data
JavaScript

// In queueEmailForRow(), getRowData() is called
// But the invoice number may have JUST been set in handleReminderStatus/handlePaidStatus
// The sheet write hasn't flushed yet, so getRowData() reads the OLD value
