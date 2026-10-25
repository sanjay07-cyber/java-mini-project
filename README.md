import java.math.BigDecimal;
import java.math.RoundingMode;
import java.time.LocalDate;
import java.time.LocalTime;
import java.time.format.DateTimeParseException;
import java.time.temporal.ChronoUnit;
import java.util.*;

// ============================================================
// TRAINER
// ============================================================

class Trainer {
    private final int id;
    private final String name;
    private final String specialization;

    Trainer(int id, String name, String specialization) {
        this.id = id;
        this.name = name;
        this.specialization = specialization;
    }

    int getId() {
        return id;
    }

    String getName() {
        return name;
    }

    String getSpecialization() {
        return specialization;
    }

    @Override
    public String toString() {
        return "T" + id + " - " + name + " (" + specialization + ")";
    }
}

// ============================================================
// PLAN
// ============================================================

class Plan {
    private final int id;
    private final String name;
    private final int durationDays;
    private final BigDecimal price;
    private final BigDecimal taxPercent;
    private final int graceDays;
    private final BigDecimal latePenalty;

    Plan(int id, String name, int durationDays,
         BigDecimal price, BigDecimal taxPercent,
         int graceDays, BigDecimal latePenalty) {

        this.id = id;
        this.name = name;
        this.durationDays = durationDays;
        this.price = price;
        this.taxPercent = taxPercent;
        this.graceDays = graceDays;
        this.latePenalty = latePenalty;
    }

    int getId() {
        return id;
    }

    String getName() {
        return name;
    }

    int getDurationDays() {
        return durationDays;
    }

    BigDecimal getPrice() {
        return price;
    }

    BigDecimal getTaxPercent() {
        return taxPercent;
    }

    int getGraceDays() {
        return graceDays;
    }

    BigDecimal getLatePenalty() {
        return latePenalty;
    }

    String cycleLabel() {

        String base;

        switch (durationDays) {

            case 30:
                base = "Monthly";
                break;

            case 90:
                base = "Quarterly";
                break;

            case 180:
                base = "Half-yearly";
                break;

            case 365:
                base = "Annual";
                break;

            default:
                base = "Custom";
                break;
        }

        return base + " (" + durationDays + " days)";
    }

    @Override
    public String toString() {

        return String.format(
                "P%d - %s | %s | Rs %s | tax %s%% | grace %d days | late fee Rs %s",
                id,
                name,
                cycleLabel(),
                price,
                taxPercent.stripTrailingZeros().toPlainString(),
                graceDays,
                latePenalty
        );
    }
}

// ============================================================
// SUBSCRIPTION
// ============================================================

class Subscription {

    enum Status {
        ACTIVE,
        EXPIRED,
        UPCOMING
    }

    private final int id;
    private final Member member;
    private final Plan plan;
    private final LocalDate startDate;
    private final LocalDate endDate;
    private final BigDecimal penalty;
    private final String type;

    private boolean invoiced = false;

    Subscription(
            int id,
            Member member,
            Plan plan,
            LocalDate startDate,
            LocalDate endDate,
            BigDecimal penalty,
            String type) {

        this.id = id;
        this.member = member;
        this.plan = plan;
        this.startDate = startDate;
        this.endDate = endDate;
        this.penalty = penalty;
        this.type = type;
    }

    int getId() {
        return id;
    }

    Member getMember() {
        return member;
    }

    Plan getPlan() {
        return plan;
    }

    LocalDate getStartDate() {
        return startDate;
    }

    LocalDate getEndDate() {
        return endDate;
    }

    BigDecimal getPenalty() {
        return penalty;
    }

    String getType() {
        return type;
    }

    boolean isInvoiced() {
        return invoiced;
    }

    void markInvoiced() {
        this.invoiced = true;
    }

    boolean covers(LocalDate d) {
        return !d.isBefore(startDate)
                && !d.isAfter(endDate);
    }

    Status statusOn(LocalDate d) {

        if (d.isBefore(startDate))
            return Status.UPCOMING;

        if (d.isAfter(endDate))
            return Status.EXPIRED;

        return Status.ACTIVE;
    }

    String describe(LocalDate asOf) {

        return String.format(
                "S%d | %s | %s | valid %s to %s | status %s (as of %s)",
                id,
                plan.getName(),
                type,
                startDate,
                endDate,
                statusOn(asOf),
                asOf
        );
    }
}

// ============================================================
// ATTENDANCE
// ============================================================

class AttendanceRecord {

    private final int id;
    private final Member member;
    private final LocalDate date;
    private final LocalTime time;

    AttendanceRecord(
            int id,
            Member member,
            LocalDate date,
            LocalTime time) {

        this.id = id;
        this.member = member;
        this.date = date;
        this.time = time;
    }

    int getId() {
        return id;
    }

    Member getMember() {
        return member;
    }

    LocalDate getDate() {
        return date;
    }

    LocalTime getTime() {
        return time;
    }
}

// ============================================================
// PAYMENT
// ============================================================

class Payment {

    private final int id;
    private final Invoice invoice;
    private final BigDecimal amount;
    private final LocalDate date;
    private final String method;

    Payment(
            int id,
            Invoice invoice,
            BigDecimal amount,
            LocalDate date,
            String method) {

        this.id = id;
        this.invoice = invoice;
        this.amount = amount;
        this.date = date;
        this.method = method;
    }

    int getId() {
        return id;
    }

    Invoice getInvoice() {
        return invoice;
    }

    BigDecimal getAmount() {
        return amount;
    }

    LocalDate getDate() {
        return date;
    }

    String getMethod() {
        return method;
    }
}

// ============================================================
// INVOICE
// ============================================================

class Invoice {

    enum Status {
        UNPAID,
        PARTIALLY_PAID,
        PAID
    }

    private final int id;
    private final Member member;
    private final Subscription subscription;
    private final LocalDate issueDate;

    private final BigDecimal subtotal;
    private final BigDecimal tax;
    private final BigDecimal penalty;
    private final BigDecimal total;

    private final List<Payment> payments =
            new ArrayList<>();

    Invoice(
            int id,
            Member member,
            Subscription sub,
            LocalDate issueDate) {

        this.id = id;
        this.member = member;
        this.subscription = sub;
        this.issueDate = issueDate;

        Plan p = sub.getPlan();

        this.subtotal =
                p.getPrice().setScale(2, RoundingMode.HALF_UP);

        this.tax =
                subtotal
                        .multiply(p.getTaxPercent())
                        .divide(
                                BigDecimal.valueOf(100),
                                2,
                                RoundingMode.HALF_UP
                        );

        this.penalty =
                sub.getPenalty()
                        .setScale(2, RoundingMode.HALF_UP);

        this.total =
                subtotal
                        .add(tax)
                        .add(penalty);
    }

    int getId() {
        return id;
    }

    Member getMember() {
        return member;
    }

    Subscription getSubscription() {
        return subscription;
    }

    BigDecimal getTotal() {
        return total;
    }

    List<Payment> getPayments() {
        return payments;
    }

    BigDecimal getPaid() {

        BigDecimal s =
                new BigDecimal("0.00");

        for (Payment p : payments) {
            s = s.add(p.getAmount());
        }

        return s;
    }

    BigDecimal getBalance() {
        return total.subtract(getPaid());
    }

    Status getStatus() {

        BigDecimal paid = getPaid();

        if (paid.signum() == 0)
            return Status.UNPAID;

        return paid.compareTo(total) >= 0
                ? Status.PAID
                : Status.PARTIALLY_PAID;
    }

    String code() {
        return "INV-" + id;
    }

    void print() {

        Plan p = subscription.getPlan();

        System.out.println(
                "============================================================"
        );

        System.out.println(
                "INVOICE " + code()
                        + "                      Date: "
                        + issueDate
        );

        System.out.println(
                "============================================================"
        );

        System.out.println(
                "Member        : "
                        + member.code()
                        + " - "
                        + member.getName()
        );

        System.out.println(
                "Plan          : "
                        + p.getName()
        );

        System.out.println(
                "Cycle         : "
                        + p.cycleLabel()
                        + ", "
                        + subscription.getStartDate()
                        + " to "
                        + subscription.getEndDate()
        );

        System.out.println(
                "------------------------------------------------------------"
        );

        System.out.printf(
                "Subtotal      : Rs %10s%n",
                subtotal
        );

        System.out.printf(
                "Tax (%s%%)     : Rs %10s%n",
                p.getTaxPercent()
                        .stripTrailingZeros()
                        .toPlainString(),
                tax
        );

        if (penalty.signum() > 0) {

            System.out.printf(
                    "Late penalty  : Rs %10s  (%s)%n",
                    penalty,
                    subscription.getType()
            );
        }

        System.out.println(
                "------------------------------------------------------------"
        );

        System.out.printf(
                "TOTAL         : Rs %10s%n",
                total
        );

        System.out.printf(
                "Paid so far   : Rs %10s%n",
                getPaid()
        );

        System.out.printf(
                "Balance due   : Rs %10s%n",
                getBalance()
        );

        System.out.println(
                "Payment status: " + getStatus()
        );

        System.out.println(
                "============================================================"
        );
    }
}

// ============================================================
// MEMBER
// ============================================================

class Member {

    private final int id;
    private final String name;
    private final String phone;
    private final String email;
    private final Trainer trainer;

    private final List<Subscription> subscriptions =
            new ArrayList<>();

    private final List<AttendanceRecord> attendance =
            new ArrayList<>();

    private final List<Invoice> invoices =
            new ArrayList<>();

    Member(
            int id,
            String name,
            String phone,
            String email,
            Trainer trainer) {

        this.id = id;
        this.name = name;
        this.phone = phone;
        this.email = email;
        this.trainer = trainer;
    }

    int getId() {
        return id;
    }

    String code() {
        return "M" + id;
    }

    String getName() {
        return name;
    }

    String getPhone() {
        return phone;
    }

    String getEmail() {
        return email;
    }

    Trainer getTrainer() {
        return trainer;
    }

    List<Subscription> getSubscriptions() {
        return subscriptions;
    }

    List<AttendanceRecord> getAttendance() {
        return attendance;
    }

    List<Invoice> getInvoices() {
        return invoices;
    }

    Subscription latestSubscription() {

        return subscriptions
                .stream()
                .max(
                        Comparator.comparing(
                                Subscription::getEndDate
                        )
                )
                .orElse(null);
    }

    Subscription subscriptionCovering(
            LocalDate d) {

        for (Subscription s : subscriptions) {

            if (s.covers(d))
                return s;
        }

        return null;
    }

    boolean hasAttendedOn(LocalDate d) {

        for (AttendanceRecord r : attendance) {

            if (r.getDate().equals(d))
                return true;
        }

        return false;
    }

    List<Subscription> uninvoicedSubscriptions() {

        List<Subscription> out =
                new ArrayList<>();

        for (Subscription s : subscriptions) {

            if (!s.isInvoiced())
                out.add(s);
        }

        return out;
    }

    BigDecimal totalDue() {

        BigDecimal due =
                new BigDecimal("0.00");

        for (Invoice i : invoices) {

            due = due.add(
                    i.getBalance()
            );
        }

        return due;
    }
}

// ============================================================
// GYM SERVICE
// ============================================================

class GymService {

    private final Map<Integer, Member> members =
            new LinkedHashMap<>();

    private final Map<Integer, Plan> plans =
            new LinkedHashMap<>();

    private final Map<Integer, Trainer> trainers =
            new LinkedHashMap<>();

    private final Map<Integer, Invoice> invoices =
            new LinkedHashMap<>();

    private int nextMemberId = 1001;
    private int nextPlanId = 1;
    private int nextTrainerId = 1;
    private int nextSubId = 1;
    private int nextInvoiceId = 5001;
    private int nextPaymentId = 1;
    private int nextAttendanceId = 1;

    Collection<Member> getMembers() {
        return members.values();
    }

    Collection<Plan> getPlans() {
        return plans.values();
    }

    Collection<Trainer> getTrainers() {
        return trainers.values();
    }

    Collection<Invoice> getInvoices() {
        return invoices.values();
    }

    Member findMember(int id) {
        return members.get(id);
    }

    Plan findPlan(int id) {
        return plans.get(id);
    }

    Trainer findTrainer(int id) {
        return trainers.get(id);
    }

    Invoice findInvoice(int id) {
        return invoices.get(id);
    }

    Trainer addTrainer(
            String name,
            String specialization) {

        Trainer t =
                new Trainer(
                        nextTrainerId++,
                        name,
                        specialization
                );

        trainers.put(t.getId(), t);

        return t;
    }

    Plan addPlan(
            String name,
            int days,
            BigDecimal price,
            BigDecimal taxPct,
            int grace,
            BigDecimal penalty) {

        if (days <= 0)
            throw new IllegalArgumentException(
                    "Duration must be positive."
            );

        if (price.signum() < 0
                || taxPct.signum() < 0
                || penalty.signum() < 0
                || grace < 0) {

            throw new IllegalArgumentException(
                    "Price, tax, grace days and penalty cannot be negative."
            );
        }

        Plan p =
                new Plan(
                        nextPlanId++,
                        name,
                        days,
                        price,
                        taxPct,
                        grace,
                        penalty
                );

        plans.put(p.getId(), p);

        return p;
    }

    Member addMember(
            String name,
            String phone,
            String email,
            Trainer trainer) {

        Member m =
                new Member(
                        nextMemberId++,
                        name,
                        phone,
                        email,
                        trainer
                );

        members.put(m.getId(), m);

        return m;
    }

    Subscription startOrRenew(
            Member m,
            Plan plan,
            LocalDate on) {

        Subscription latest =
                m.latestSubscription();

        LocalDate start;

        BigDecimal penalty =
                BigDecimal.ZERO;

        String type;

        if (latest == null) {

            start = on;
            type = "NEW";

        } else if (!latest.getEndDate().isBefore(on)) {

            start =
                    latest.getEndDate()
                            .plusDays(1);

            type =
                    "RENEWAL (extends current validity)";

        } else {

            start = on;

            long daysLate =
                    ChronoUnit.DAYS.between(
                            latest.getEndDate(),
                            on
                    );

            if (daysLate > plan.getGraceDays()) {

                penalty =
                        plan.getLatePenalty();

                type =
                        "LATE RENEWAL ("
                        + daysLate
                        + " days after expiry, grace "
                        + plan.getGraceDays()
                        + ")";

            } else {

                type =
                        "RENEWAL ("
                        + daysLate
                        + " days after expiry, within grace)";
            }
        }

        LocalDate end =
                start.plusDays(
                        plan.getDurationDays() - 1L
                );

        Subscription s =
                new Subscription(
                        nextSubId++,
                        m,
                        plan,
                        start,
                        end,
                        penalty,
                        type
                );

        m.getSubscriptions().add(s);

        return s;
    }

    AttendanceRecord recordAttendance(
            Member m,
            LocalDate date) {

        Subscription s =
                m.subscriptionCovering(date);

        if (s == null) {

            throw new IllegalStateException(
                    "Member "
                    + m.code()
                    + " has no active subscription on "
                    + date
                    + ". Start or renew a subscription first."
            );
        }

        if (m.hasAttendedOn(date)) {

            throw new IllegalStateException(
                    "Attendance already recorded for "
                    + m.code()
                    + " on "
                    + date
                    + "."
            );
        }

        AttendanceRecord r =
                new AttendanceRecord(
                        nextAttendanceId++,
                        m,
                        date,
                        LocalTime.now().withNano(0)
                );

        m.getAttendance().add(r);

        return r;
    }

    Invoice generateInvoice(
            Member m,
            Subscription sub,
            LocalDate date) {

        if (sub.isInvoiced()) {

            throw new IllegalStateException(
                    "Subscription S"
                    + sub.getId()
                    + " is already invoiced."
            );
        }

        Invoice inv =
                new Invoice(
                        nextInvoiceId++,
                        m,
                        sub,
                        date
                );

        sub.markInvoiced();

        m.getInvoices().add(inv);

        invoices.put(
                inv.getId(),
                inv
        );

        return inv;
    }

    Payment recordPayment(
            Invoice inv,
            BigDecimal amount,
            String method,
            LocalDate date) {

        if (inv == null) {

            throw new IllegalStateException(
                    "An invoice must be generated before accepting payment."
            );
        }

        amount =
                amount.setScale(
                        2,
                        RoundingMode.HALF_UP
                );

        if (amount.signum() <= 0) {

            throw new IllegalArgumentException(
                    "Payment amount must be greater than zero."
            );
        }

        if (inv.getBalance().signum() <= 0) {

            throw new IllegalStateException(
                    inv.code()
                    + " is already fully paid."
            );
        }

        if (amount.compareTo(
                inv.getBalance()) > 0) {

            throw new IllegalArgumentException(
                    "Amount exceeds balance due (Rs "
                    + inv.getBalance()
                    + ")."
            );
        }

        Payment p =
                new Payment(
                        nextPaymentId++,
                        inv,
                        amount,
                        date,
                        method
                );

        inv.getPayments().add(p);

        return p;
    }
}

// ============================================================
// MAIN CLASS
// ============================================================

public class Main {

    static Scanner sc =
            new Scanner(System.in);

    static GymService gym =
            new GymService();

    public static void main(String[] args) {

        // ----------------------------------------------------
        // SAMPLE TRAINERS
        // ----------------------------------------------------

        Trainer t1 =
                gym.addTrainer(
                        "Arun",
                        "Weight Training"
                );

        Trainer t2 =
                gym.addTrainer(
                        "Rahul",
                        "Cardio"
                );

        // ----------------------------------------------------
        // SAMPLE PLANS
        // ----------------------------------------------------

        Plan p1 =
                gym.addPlan(
                        "Basic",
                        30,
                        new BigDecimal("1000"),
                        new BigDecimal("18"),
                        5,
                        new BigDecimal("100")
                );

        Plan p2 =
                gym.addPlan(
                        "Premium",
                        90,
                        new BigDecimal("2500"),
                        new BigDecimal("18"),
                        7,
                        new BigDecimal("200")
                );

        // ----------------------------------------------------
        // SAMPLE MEMBERS
        // ----------------------------------------------------

        Member m1 =
                gym.addMember(
                        "Sanjay",
                        "9876543210",
                        "sanjay@gmail.com",
                        t1
                );

        Member m2 =
                gym.addMember(
                        "Vishva",
                        "9876501234",
                        "vishva@gmail.com",
                        t2
                );

        // ----------------------------------------------------
        // SAMPLE SUBSCRIPTIONS
        // ----------------------------------------------------

        gym.startOrRenew(
                m1,
                p1,
                LocalDate.now()
        );

        gym.startOrRenew(
                m2,
                p2,
                LocalDate.now()
        );

        menu();
    }

    // ========================================================
    // MENU
    // ========================================================

    static void menu() {

        while (true) {

            System.out.println();
            System.out.println(
                    "================================================"
            );

            System.out.println(
                    "       GYM MEMBERSHIP & BILLING SYSTEM"
            );

            System.out.println(
                    "================================================"
            );

            System.out.println(
                    "1.  Add Trainer"
            );

            System.out.println(
                    "2.  View Trainers"
            );

            System.out.println(
                    "3.  Add Plan"
            );

            System.out.println(
                    "4.  View Plans"
            );

            System.out.println(
                    "5.  Add Member"
            );

            System.out.println(
                    "6.  View Members"
            );

            System.out.println(
                    "7.  Start / Renew Membership"
            );

            System.out.println(
                    "8.  View Subscriptions"
            );

            System.out.println(
                    "9.  Record Attendance"
            );

            System.out.println(
                    "10. Generate Invoice"
            );

            System.out.println(
                    "11. View Invoices"
            );

            System.out.println(
                    "12. Make Payment"
            );

            System.out.println(
                    "13. View Member Details"
            );

            System.out.println(
                    "14. Exit"
            );

            System.out.println(
                    "================================================"
            );

            int choice =
                    readInt(
                            "Enter your choice: "
                    );

            try {

                switch (choice) {

                    case 1:
                        addTrainer();
                        break;

                    case 2:
                        viewTrainers();
                        break;

                    case 3:
                        addPlan();
                        break;

                    case 4:
                        viewPlans();
                        break;

                    case 5:
                        addMember();
                        break;

                    case 6:
                        viewMembers();
                        break;

                    case 7:
                        startRenewMembership();
                        break;

                    case 8:
                        viewSubscriptions();
                        break;

                    case 9:
                        recordAttendance();
                        break;

                    case 10:
                        generateInvoice();
                        break;

                    case 11:
                        viewInvoices();
                        break;

                    case 12:
                        makePayment();
                        break;

                    case 13:
                        viewMemberDetails();
                        break;

                    case 14:
                        System.out.println(
                                "Thank you for using Gym Membership System!"
                        );

                        sc.close();
                        return;

                    default:
                        System.out.println(
                                "Invalid choice."
                        );
                }

            } catch (Exception e) {

                System.out.println();
                System.out.println(
                        "ERROR: " + e.getMessage()
                );
            }
        }
    }

    // ========================================================
    // ADD TRAINER
    // ========================================================

    static void addTrainer() {

        System.out.println(
                "\n--- ADD TRAINER ---"
        );

        System.out.print(
                "Enter trainer name: "
        );

        String name =
                sc.nextLine();

        System.out.print(
                "Enter specialization: "
        );

        String specialization =
                sc.nextLine();

        Trainer t =
                gym.addTrainer(
                        name,
                        specialization
                );

        System.out.println(
                "\nTrainer added successfully!"
        );

        System.out.println(t);
    }

    // ========================================================
    // VIEW TRAINERS
    // ========================================================

    static void viewTrainers() {

        System.out.println(
                "\n--- TRAINERS ---"
        );

        if (gym.getTrainers().isEmpty()) {

            System.out.println(
                    "No trainers available."
            );

            return;
        }

        for (Trainer t :
                gym.getTrainers()) {

            System.out.println(t);
        }
    }

    // ========================================================
    // ADD PLAN
    // ========================================================

    static void addPlan() {

        System.out.println(
                "\n--- ADD PLAN ---"
        );

        System.out.print(
                "Enter plan name: "
        );

        String name =
                sc.nextLine();

        int days =
                readInt(
                        "Enter duration in days: "
                );

        BigDecimal price =
                readDecimal(
                        "Enter price: "
                );

        BigDecimal tax =
                readDecimal(
                        "Enter tax percentage: "
                );

        int grace =
                readInt(
                        "Enter grace days: "
                );

        BigDecimal penalty =
                readDecimal(
                        "Enter late penalty: "
                );

        Plan p =
                gym.addPlan(
                        name,
                        days,
                        price,
                        tax,
                        grace,
                        penalty
                );

        System.out.println(
                "\nPlan added successfully!"
        );

        System.out.println(p);
    }

    // ========================================================
    // VIEW PLANS
    // ========================================================

    static void viewPlans() {

        System.out.println(
                "\n--- PLANS ---"
        );

        if (gym.getPlans().isEmpty()) {

            System.out.println(
                    "No plans available."
            );

            return;
        }

        for (Plan p :
                gym.getPlans()) {

            System.out.println(p);
        }
    }

    // ========================================================
    // ADD MEMBER
    // ========================================================

    static void addMember() {

        System.out.println(
                "\n--- ADD MEMBER ---"
        );

        System.out.print(
                "Enter member name: "
        );

        String name =
                sc.nextLine();

        System.out.print(
                "Enter phone: "
        );

        String phone =
                sc.nextLine();

        System.out.print(
                "Enter email: "
        );

        String email =
                sc.nextLine();

        viewTrainers();

        int trainerId =
                readInt(
                        "Enter trainer ID (0 for none): "
                );

        Trainer trainer = null;

        if (trainerId != 0) {

            trainer =
                    gym.findTrainer(
                            trainerId
                    );

            if (trainer == null) {

                System.out.println(
                        "Trainer not found."
                );

                return;
            }
        }

        Member m =
                gym.addMember(
                        name,
                        phone,
                        email,
                        trainer
                );

        System.out.println(
                "\nMember added successfully!"
        );

        System.out.println(
                "Member ID: " + m.code()
        );
    }

    // ========================================================
    // VIEW MEMBERS
    // ========================================================

    static void viewMembers() {

        System.out.println(
                "\n--- MEMBERS ---"
        );

        if (gym.getMembers().isEmpty()) {

            System.out.println(
                    "No members available."
            );

            return;
        }

        for (Member m :
                gym.getMembers()) {

            System.out.println(
                    "----------------------------------------"
            );

            System.out.println(
                    "ID      : " + m.code()
            );

            System.out.println(
                    "Name    : " + m.getName()
            );

            System.out.println(
                    "Phone   : " + m.getPhone()
            );

            System.out.println(
                    "Email   : " + m.getEmail()
            );

            if (m.getTrainer() != null) {

                System.out.println(
                        "Trainer : "
                        + m.getTrainer().getName()
                );

            } else {

                System.out.println(
                        "Trainer : None"
                );
            }
        }
    }

    // ========================================================
    // START / RENEW MEMBERSHIP
    // ========================================================

    static void startRenewMembership() {

        System.out.println(
                "\n--- START / RENEW MEMBERSHIP ---"
        );

        int memberId =
                readInt(
                        "Enter member ID: "
                );

        Member member =
                gym.findMember(
                        memberId
                );

        if (member == null) {

            System.out.println(
                    "Member not found."
            );

            return;
        }

        viewPlans();

        int planId =
                readInt(
                        "Enter plan ID: "
                );

        Plan plan =
                gym.findPlan(
                        planId
                );

        if (plan == null) {

            System.out.println(
                    "Plan not found."
            );

            return;
        }

        LocalDate date =
                readDate(
                        "Enter start/renewal date (YYYY-MM-DD): "
                );

        Subscription sub =
                gym.startOrRenew(
                        member,
                        plan,
                        date
                );

        System.out.println(
                "\nMembership created successfully!"
        );

        System.out.println(
                sub.describe(date)
        );
    }

    // ========================================================
    // VIEW SUBSCRIPTIONS
    // ========================================================

    static void viewSubscriptions() {

        System.out.println(
                "\n--- SUBSCRIPTIONS ---"
        );

        for (Member m :
                gym.getMembers()) {

            if (m.getSubscriptions().isEmpty())
                continue;

            System.out.println();

            System.out.println(
                    "Member: "
                    + m.code()
                    + " - "
                    + m.getName()
            );

            for (Subscription s :
                    m.getSubscriptions()) {

                System.out.println(
                        s.describe(
                                LocalDate.now()
                        )
                );
            }
        }
    }

    // ========================================================
    // ATTENDANCE
    // ========================================================

    static void recordAttendance() {

        System.out.println(
                "\n--- RECORD ATTENDANCE ---"
        );

        int memberId =
                readInt(
                        "Enter member ID: "
                );

        Member member =
                gym.findMember(
                        memberId
                );

        if (member == null) {

            System.out.println(
                    "Member not found."
            );

            return;
        }

        LocalDate date =
                readDate(
                        "Enter attendance date (YYYY-MM-DD): "
                );

        AttendanceRecord record =
                gym.recordAttendance(
                        member,
                        date
                );

        System.out.println(
                "\nAttendance recorded successfully!"
        );

        System.out.println(
                "Attendance ID : "
                + record.getId()
        );

        System.out.println(
                "Member        : "
                + member.code()
        );

        System.out.println(
                "Date          : "
                + record.getDate()
        );

        System.out.println(
                "Time          : "
                + record.getTime()
        );
    }

    // ========================================================
    // GENERATE INVOICE
    // ========================================================

    static void generateInvoice() {

        System.out.println(
                "\n--- GENERATE INVOICE ---"
        );

        int memberId =
                readInt(
                        "Enter member ID: "
                );

        Member member =
                gym.findMember(
                        memberId
                );

        if (member == null) {

            System.out.println(
                    "Member not found."
            );

            return;
        }

        List<Subscription> subscriptions =
                member.uninvoicedSubscriptions();

        if (subscriptions.isEmpty()) {

            System.out.println(
                    "No uninvoiced subscriptions found."
            );

            return;
        }

        System.out.println(
                "\nUninvoiced Subscriptions:"
        );

        for (Subscription s :
                subscriptions) {

            System.out.println(
                    s.describe(
                            LocalDate.now()
                    )
            );
        }

        int subId =
                readInt(
                        "Enter subscription ID to invoice: "
                );

        Subscription selected = null;

        for (Subscription s :
                subscriptions) {

            if (s.getId() == subId) {

                selected = s;
                break;
            }
        }

        if (selected == null) {

            System.out.println(
                    "Subscription not found."
            );

            return;
        }

        Invoice invoice =
                gym.generateInvoice(
                        member,
                        selected,
                        LocalDate.now()
                );

        System.out.println(
                "\nInvoice generated successfully!"
        );

        invoice.print();
    }

    // ========================================================
    // VIEW INVOICES
    // ========================================================

    static void viewInvoices() {

        System.out.println(
                "\n--- INVOICES ---"
        );

        if (gym.getInvoices().isEmpty()) {

            System.out.println(
                    "No invoices available."
            );

            return;
        }

        for (Invoice invoice :
                gym.getInvoices()) {

            System.out.println(
                    invoice.code()
                    + " | "
                    + invoice.getMember().getName()
                    + " | Total: Rs "
                    + invoice.getTotal()
                    + " | Paid: Rs "
                    + invoice.getPaid()
                    + " | Balance: Rs "
                    + invoice.getBalance()
                    + " | Status: "
                    + invoice.getStatus()
            );
        }
    }

    // ========================================================
    // PAYMENT
    // ========================================================

    static void makePayment() {

        System.out.println(
                "\n--- MAKE PAYMENT ---"
        );

        int invoiceId =
                readInt(
                        "Enter invoice ID: "
                );

        Invoice invoice =
                gym.findInvoice(
                        invoiceId
                );

        if (invoice == null) {

            System.out.println(
                    "Invoice not found."
            );

            return;
        }

        invoice.print();

        BigDecimal amount =
                readDecimal(
                        "Enter payment amount: "
                );

        System.out.print(
                "Enter payment method (Cash/UPI/Card): "
        );

        String method =
                sc.nextLine();

        Payment payment =
                gym.recordPayment(
                        invoice,
                        amount,
                        method,
                        LocalDate.now()
                );

        System.out.println(
                "\nPayment successful!"
        );

        System.out.println(
                "Payment ID : "
                + payment.getId()
        );

        System.out.println(
                "Amount     : Rs "
                + payment.getAmount()
        );

        System.out.println(
                "Method     : "
                + payment.getMethod()
        );

        System.out.println(
                "Balance    : Rs "
                + invoice.getBalance()
        );

        System.out.println(
                "Status     : "
                + invoice.getStatus()
        );
    }

    // ========================================================
    // MEMBER DETAILS
    // ========================================================

    static void viewMemberDetails() {

        System.out.println(
                "\n--- MEMBER DETAILS ---"
        );

        int memberId =
                readInt(
                        "Enter member ID: "
                );

        Member m =
                gym.findMember(
                        memberId
                );

        if (m == null) {

            System.out.println(
                    "Member not found."
            );

            return;
        }

        System.out.println(
                "\n================================"
        );

        System.out.println(
                "        MEMBER DETAILS"
        );

        System.out.println(
                "================================"
        );

        System.out.println(
                "Member ID : " + m.code()
        );

        System.out.println(
                "Name      : " + m.getName()
        );

        System.out.println(
                "Phone     : " + m.getPhone()
        );

        System.out.println(
                "Email     : " + m.getEmail()
        );

        if (m.getTrainer() != null) {

            System.out.println(
                    "Trainer   : "
                    + m.getTrainer()
            );

        } else {

            System.out.println(
                    "Trainer   : None"
            );
        }

        System.out.println(
                "\n--- SUBSCRIPTIONS ---"
        );

        if (m.getSubscriptions().isEmpty()) {

            System.out.println(
                    "No subscriptions."
            );

        } else {

            for (Subscription s :
                    m.getSubscriptions()) {

                System.out.println(
                        s.describe(
                                LocalDate.now()
                        )
                );
            }
        }

        System.out.println(
                "\n--- ATTENDANCE ---"
        );

        if (m.getAttendance().isEmpty()) {

            System.out.println(
                    "No attendance records."
            );

        } else {

            for (AttendanceRecord r :
                    m.getAttendance()) {

                System.out.println(
                        "Attendance ID: "
                        + r.getId()
                        + " | Date: "
                        + r.getDate()
                        + " | Time: "
                        + r.getTime()
                );
            }
        }

        System.out.println(
                "\n--- INVOICES ---"
        );

        if (m.getInvoices().isEmpty()) {

            System.out.println(
                    "No invoices."
            );

        } else {

            for (Invoice i :
                    m.getInvoices()) {

                System.out.println(
                        i.code()
                        + " | Total: Rs "
                        + i.getTotal()
                        + " | Balance: Rs "
                        + i.getBalance()
                        + " | "
                        + i.getStatus()
                );
            }
        }

        System.out.println(
                "\nTotal Amount Due: Rs "
                + m.totalDue()
        );
    }

    // ========================================================
    // INPUT FUNCTIONS
    // ========================================================

    static int readInt(String message) {

        while (true) {

            try {

                System.out.print(message);

                String input =
                        sc.nextLine();

                return Integer.parseInt(
                        input.trim()
                );

            } catch (NumberFormatException e) {

                System.out.println(
                        "Please enter a valid number."
                );
            }
        }
    }

    static BigDecimal readDecimal(
            String message) {

        while (true) {

            try {

                System.out.print(message);

                String input =
                        sc.nextLine();

                return new BigDecimal(
                        input.trim()
                );

            } catch (NumberFormatException e) {

                System.out.println(
                        "Please enter a valid amount."
                );
            }
        }
    }

    static LocalDate readDate(
            String message) {

        while (true) {

            try {

                System.out.print(message);

                String input =
                        sc.nextLine();

                return LocalDate.parse(
                        input.trim()
                );

            } catch (DateTimeParseException e) {

                System.out.println(
                        "Invalid date. Use YYYY-MM-DD."
                );
            }
        }
    }
}
